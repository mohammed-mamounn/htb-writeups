## Introduction

Quantum Key Distribution (QKD) is a method of establishing a cryptographic key between two parties using the laws of quantum mechanics rather than mathematical complexity. The most well-known protocol is BB84, which encodes each bit of the key into the quantum state of a single photon. The security guarantee comes from a fundamental property of quantum physics — measuring a quantum state disturbs it irreversibly. This means any eavesdropper trying to intercept the key leaves a detectable trace in the error rate, and the two legitimate parties can detect the intrusion and abort.

This challenge simulates a QKD network with three parties: a Transmitter (TX), a Receiver (RX), and a Trusted Node sitting between them — which is us. The goal is to fully recover the shared key that TX and RX establish, then use it to decrypt the flag from the Receiver.

---

## Methodology

I started by downloading the challenge archive which contained four source files: `server.py`, `transmitter.py`, `receiver.py`, and `util.py`. Since this is a code analysis challenge, I read every file before touching the network.

`server.py` orchestrates the session. It instantiates a Transmitter and Receiver, runs the QKD exchange through us (the Trusted Node), and if the key is successfully established, enters a command loop where data sent to the Receiver is decrypted and executed as a command. The only command that returns the flag is `TX|FETCH|SECRET`.

`transmitter.py` is where I found the vulnerability.

---

## Finding the Vulnerability

Reading the Transmitter's `generate_circuits()` method, I noticed this:

```python
k = np.random.poisson(self.λ) + 2
for _ in range(k):
    circuit = QuantumCircuit(1, 1)
    ...
    circuits.append(circuit)
```

For every single bit of the key, the Transmitter generates not one qubit but **k copies**, where k is drawn from a Poisson distribution with λ=2, plus a minimum of 2. So every bit has at least 2 identical quantum copies, with an average of 4.

The changelog in the source confirmed this was intentional — a "legacy sync signal" feature added in v1.1.0 for backward compatibility. The k-values are also handed to the Trusted Node in plaintext at the start of every session as the "sync signal."

This is a textbook **Photon Number Splitting (PNS) attack**. In real QKD, the no-cloning theorem guarantees that a single photon cannot be copied or measured without disturbing it. But when the transmitter leaks multiple copies of the same qubit, a man-in-the-middle can peel off spare copies and measure them freely — the remaining copies still reach the Receiver undisturbed, so neither party detects anything wrong.

The multi-photon emission completely breaks the quantum security guarantee.

---

## Exploitation

I wrote a Python exploit using `pwntools` to interact with the server programmatically.

### Phase 1 — Intercepting Qubits

The Trusted Node gets to specify a measurement gate for every qubit before they reach the Receiver. Gates are `0` (Z-basis), `1` (X-basis), or `-1` (pass through untouched).

Since I knew the k-values from the sync signal, my strategy was:

- Position 0 of each chunk → gate `0` (measure in Z-basis, record result)
- Position 1 of each chunk → gate `1` (measure in X-basis, record result)
- Positions 2 through k-1 → gate `-1` (forward to Receiver untouched)

```python
gates = []
for k in sync_signal:
    gates.extend([0, 1])
    if k > 2:
        gates.extend([-1] * (k - 2))
```

BB84 uses two measurement bases and TX randomly picks one per bit. By measuring in both Z and X, one of my two results is guaranteed to match TX's choice. I stored both results per chunk and waited for the server to return the measurement outcomes. The Receiver meanwhile got k−2 qubits per bit, completely unaware anything had been intercepted.

### Phase 2 — Manipulating Reconciliation

After measurements, TX and RX perform classical reconciliation — they compare which bases they used and keep only the bits where they matched. As the Trusted Node I also get to report which gates the Receiver supposedly used, and this is where I manipulated the outcome.

For the two positions I measured myself, I reported gate value `2` — which is impossible in the real protocol (only 0 and 1 are valid). TX's own gates are only ever 0 or 1, so TX can never find a match at those positions. They get silently excluded from the key without raising any error.

For all passthrough positions I forwarded to the Receiver, I reported Bob's real gate values verbatim. TX reconciles normally with RX as if I was never there.

```python
sub = []
idx = 0
for k in sync_signal:
    sub.extend([2, 2])
    idx += 2
    for _ in range(k - 2):
        sub.append(rx_gates[idx])
        idx += 1
```

### Key Recovery

TX reports back which positions it found basis matches at. Every matched position is a passthrough qubit, and the Receiver's gate at that position equals TX's basis by definition. I looked up my measurement in that same basis and recorded the bit:

```python
for ci, k in enumerate(sync_signal):
    for _ in range(k):
        if tx_matches[idx]:
            basis = rx_gates[idx]
            key_bits.append(chunk_z[ci] if basis == 0 else chunk_x[ci])
        idx += 1
```

Both TX and RX derive the final key with `sha256(key_bits_string).digest()`. I applied the same derivation and now held the exact 32-byte key they computed.

### Getting the Flag

I XOR'd the command `TX|FETCH|SECRET` with the recovered key and sent the ciphertext to the server:

```python
key = hashlib.sha256("".join(key_bits).encode()).digest()
payload = xor(b"TX|FETCH|SECRET", key).hex()
```

The Receiver decrypted it, matched the command, and returned the flag.

---

## What I Learned

- Real-world QKD security depends entirely on single-photon emission. The moment a transmitter leaks multiple photons per bit, the no-cloning theorem no longer protects the channel and a PNS attack becomes trivial.
- "Legacy compatibility" features in security-critical protocols are dangerous. The multi-photon sync signal was added for backward compatibility but silently destroyed the entire security model.
- Quantum cryptography is not magic — its security assumptions are precise and fragile. Breaking one physical assumption (single photon per pulse) breaks the whole protocol, regardless of how correct the classical reconciliation logic is.
- Source code review was the entire attack surface here. The vulnerability was not in a service or a binary — it was in a design decision visible only by reading the implementation.

---

## Summary of the Full Chain

1. Downloaded source files — read `transmitter.py`, identified k≥2 copies per bit
2. Recognised the attack pattern as Photon Number Splitting (PNS)
3. Connected to the server with a `pwntools` script
4. Parsed the plaintext sync signal to know k-values for all 128 bits
5. **Phase 1**: Sent gates to measure positions 0 (Z-basis) and 1 (X-basis) per chunk, forward the rest
6. Recorded both measurement results per bit from the server's response
7. **Phase 2**: Substituted garbage gates (value 2) for our measured positions, Bob's real gates for passthroughs
8. Parsed TX's match list to identify which bits entered the key
9. Reconstructed the raw key bitstring using our stored measurements
10. Applied SHA-256 to derive the final session key
11. XOR'd `TX|FETCH|SECRET` with the key → sent ciphertext → flag captured

---

## Remediation

- **Enforce single-photon emission** — k must equal 1 for every bit. The multi-photon sync signal must be removed entirely, regardless of legacy compatibility concerns.
- **Implement decoy-state QKD** — in real optical systems where true single-photon sources are impractical, decoy states allow TX and RX to statistically detect PNS attacks by monitoring channel transmittance anomalies.
- **Never expose k-values** — even if multi-photon pulses were unavoidable, providing the sync signal in plaintext to the Trusted Node eliminates any remaining ambiguity for an attacker.
- **Treat the Trusted Node as untrusted** — the protocol architecture assumed the middle node was honest. Any production QKD system should be designed so that even a fully compromised relay node cannot recover the key.
