# Redeemer | Machine | Very Easy | Linux

## Introduction

Redis is a database that is a database that ended up managed in the main memory of the system. The Redis server is referred to as one of the main memory ("in-memory DBs") like these because they are frequently used to cache data that is frequently requested. Redis is used/requested for quicker retrieval. Redis is composed of a server, CLI, & db running in RAM.

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

## Connecting to Redis (completed / researched)

*My notes stopped right after installing `redis-tools`. I looked up the rest of the Redeemer box and finished the write-up below.*

Redis is unauthenticated by default in this box, so a plain connection with the `-h` flag is enough:

```
redis-cli -h <target-ip>
```

Once connected, the `INFO` command dumps server statistics, including the version:

```
INFO
```

This confirms the server is running **Redis 5.0.7**.

Redis can hold multiple numbered databases. To see how many keys live in database 0:

```
SELECT 0
KEYS *
```

This returned 4 keys, one of which is named `flag`. Reading it out:

```
GET flag
```

This returns the flag directly — Redeemer doesn't have a separate user/root split, since Redis (running as root, with no authentication) hands over the flag as soon as you can talk to it.

### Summary of the full chain
1. Full port scan (`nmap -p- -sV`) → found Redis on TCP 6379 (default full-range/default scan missed it)
2. Installed `redis-tools` to get `redis-cli`
3. Connected unauthenticated with `redis-cli -h <target-ip>`
4. `INFO` → confirmed Redis 5.0.7, unauthenticated access
5. `SELECT 0` → `KEYS *` → found a key named `flag`
6. `GET flag` → flag captured

### Remediation
- Require authentication (`requirepass`) on the Redis instance.
- Bind Redis to localhost/internal interfaces only — don't expose 6379 to untrusted networks.
- Don't store secrets/flags in plaintext inside a database with no access control.
