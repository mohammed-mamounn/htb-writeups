# htb-writeups

Documentation of my journey through different HTB machines and challenges.

## Machines

| Machine | OS | Difficulty | Key Technique |
|---|---|---|---|
| [Vaccine](./Vaccine) | Linux | Very Easy | SQL injection → ```sqlmap --os-shell``` for RCE, then GTFOBins sudo vi privilege escalation |
| [Redeemer](./Redeemer) | Linux | Very Easy | Unauthenticated Redis access |
| [Fawn](./Fawn) | Linux | Very Easy | Anonymous FTP login |
| [Oopsie](./Oopsie) (Advanced) | Linux | Very Easy | Cookie value manipulation (user_id) → PHP webshell upload → PATH hijack on SUID binary |


## Challenges
| Challenge | Type | Difficulty | Key Technique |
|---|---|---|---|
| [Untrusted Node](./Untrusted_Node) | Quantum | Medium | Photon Number Splitting attack on BB84 QKD → key extraction |
