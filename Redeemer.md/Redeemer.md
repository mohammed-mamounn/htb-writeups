# Redeemer | Machine | Very Easy | Linux

## Introduction

Redis is a database that lives entirely in the main memory of the system — it's referred to as an "in-memory DB" for this reason. It's frequently used to cache data that needs to be requested and retrieved quickly. Redis is composed of a server, a CLI, & a database that runs in RAM.

## Methodology

I started by enumerating using nmap, however the usual `nmap -sV` did not provide any open ports.

Some research showed that the usual nmap command only scans the 1000 most common TCP ports, whether TCP or UDP is specified.

Next, I used `-p-` to scan all 65,535 ports:

```
nmap -sV -p- <target>
```

That showed an open port: 6379, redis.

Next, I installed `redis-tools` via `sudo apt install redis-tools`, to be able to interact with it.

After installation I used `redis-cli --help`

to view the available flags & how to connect to a remote Redis server.

---

## Connecting to Redis


Redis is unauthenticated by default in this box, so a plain connection with the `-h` flag is enough:

```
redis-cli -h <target-ip>
```

Once connected, the `INFO` command showed me server statistics, including the version:

```
INFO
```

This confirms the server is running **Redis 5.0.7**.
I tried looking online for some known vulnerabilities for this specific version of Redis but no luck.

Redis can hold multiple numbered databases, I found 1 database in position 0, next I wanted to see how many keys live in it:

```
SELECT 0
KEYS *
```

This returned 4 keys, one of which is named `flag`. ```cat``` did not cut it here as this is a databse so instead:

```
GET flag
```

This gave me the flag directly.

### Summary of the full chain
1. Full port scan (`nmap -p- -sV`) → found Redis on TCP 6379 (default full-range/default scan missed it)
2. Installed `redis-tools` to get `redis-cli`
3. Connected unauthenticated with `redis-cli -h <target-ip>`
4. `INFO` → confirmed Redis 5.0.7, unauthenticated access
5. `SELECT 0` → `KEYS *` → found a key named `flag`
6. `GET flag` → flag captured

### Remediation
- Require authentication on the Redis instance.
- Bind Redis to localhost/internal interfaces only so port 6379 is not exposed to untrusted networks.
- Don't store secrets/flags in plaintext inside a database with no access control.
