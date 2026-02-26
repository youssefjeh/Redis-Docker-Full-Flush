

# Redis Docker Full Flush Documentation

This README documents the full process of flushing a Redis instance running in Docker, including clearing both RAM and persistent storage (RDB and AOF), in a secured production configuration where `FLUSHALL`, `FLUSHDB`, and `CONFIG` commands are disabled.

---

## 1. Docker Containers Running

```bash
wfc@redis:~$ docker ps
CONTAINER ID   IMAGE              COMMAND                  CREATED         STATUS         PORTS                                                                          NAMES
6af72d12e183   cassandra:latest   "docker-entrypoint.s…"   10 months ago   Up 10 months   7000-7001/tcp, 7199/tcp, 9160/tcp, 0.0.0.0:9042->9042/tcp, :::9042->9042/tcp   my-cassandra
f1a02a84d1c9   redis:latest       "docker-entrypoint.s…"   10 months ago   Up 10 months   0.0.0.0:6379->6379/tcp, :::6379->6379/tcp                                      redis
```

---

## 2. Attempting FLUSHALL / FLUSHDB

```bash
wfc@redis:~$ docker exec -it redis redis-cli FLUSHALL
(error) ERR unknown command 'FLUSHALL', with args beginning with:
wfc@redis:~$ docker exec -it redis redis-cli FLUSHDB
(error) ERR unknown command 'FLUSHDB', with args beginning with:
```

> Commands are disabled in this secured Redis configuration.

---

## 3. Testing Redis connection

```bash
wfc@redis:~$ docker exec -it redis redis-cli PING
(error) NOAUTH Authentication required.
wfc@redis:~$ docker exec -it redis redis-cli -a 'myP@ssw00rd' PING
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
PONG
```

---

## 4. Checking FLUSHALL availability

```bash
wfc@redis:~$ docker exec -it redis redis-cli -a 'myP@ssw00rd' COMMAND INFO FLUSHALL
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
1) (nil)
```

> `FLUSHALL` is disabled.

```bash
wfc@redis:~$ docker exec -it redis redis-cli -a 'myP@ssw00rd' CONFIG GET "*flush*"
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
(error) ERR unknown command 'CONFIG', with args beginning with: 'GET' '*flush*'
```

> `CONFIG` is also disabled.

---

## 5. Checking DB size

```bash
wfc@redis:~$ docker exec redis redis-cli -a 'myP@ssw00rd' DBSIZE
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
101
```

---

## 6. Manual deletion of keys

```bash
wfc@redis:~$ docker exec redis redis-cli -a 'myP@ssw00rd' --scan | \
xargs -r -L 100 docker exec -i redis redis-cli -a 'myP@ssw00rd' DEL
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
99
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
1
```

```bash
wfc@redis:~$ docker exec redis redis-cli -a 'myP@ssw00rd' DBSIZE
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
3
```

> Most keys deleted, but persistence remains.

---

## 7. Restarting Redis (data comes back)

```bash
wfc@redis:~$ docker restart redis
redis
wfc@redis:~$ docker exec redis redis-cli -a 'myP@ssw00rd' DBSIZE
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
16
```

---

## 8. Checking persistence

```bash
wfc@redis:~$ docker exec redis redis-cli -a 'myP@ssw00rd' INFO persistence
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
# Persistence
loading:0
async_loading:0
current_cow_peak:0
current_cow_size:0
current_cow_size_age:0
current_fork_perc:0.00
current_save_keys_processed:0
current_save_keys_total:0
rdb_changes_since_last_save:323
rdb_bgsave_in_progress:0
rdb_last_save_time:1772103718
rdb_last_bgsave_status:ok
rdb_last_save_time_sec:0
rdb_current_bgsave_time_sec:-1
rdb_saves:1
rdb_last_cow_size:897024
rdb_last_load_keys_expired:0
rdb_last_load_keys_loaded:19
aof_enabled:1
...
```

> AOF enabled, RDB snapshot present.

---

## 9. Entering container and inspecting /data

```bash
wfc@redis:~$ docker exec -it redis bash
root@f1a02a84d1c9:/data# ls -lah
total 20K
drwxr-xr-x 3 redis redis 4.0K Feb 26 11:01 .
drwxr-xr-x 1 root  root  4.0K Apr 29  2025 ..
drwx------ 2 redis redis 4.0K Feb 25 14:56 appendonlydir
-rw------- 1 redis redis 6.9K Feb 26 11:01 dump.rdb
```

---

## 10. Removing persistent files

```bash
root@f1a02a84d1c9:/data# rm -f /data/dump.rdb
root@f1a02a84d1c9:/data# rm -rf /data/appendonlydir/*
root@f1a02a84d1c9:/data# ls -lah /data
total 12K
drwxr-xr-x 3 redis redis 4.0K Feb 26 11:06 .
drwxr-xr-x 1 root  root  4.0K Apr 29  2025 ..
drwx------ 2 redis redis 4.0K Feb 26 11:06 appendonlydir
```

---

## 11. Restarting Redis

```bash
wfc@redis:~$ docker restart redis
redis
```

---

## 12. Verifying Redis is completely empty

```bash
wfc@redis:~$ docker exec redis redis-cli -a 'myP@ssw00rd' DBSIZE
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
0
```

✅ Redis is now **fully flushé**.
No keys in RAM. No persistent files. Full reset achieved.

---

## 13. Notes

* `FLUSHALL`, `FLUSHDB`, `CONFIG` disabled for production safety.
* Deleting `/data/dump.rdb` and `/data/appendonlydir/*` is required for a true flush.
* Restart Redis after deletion to confirm.
* Optional: disable persistence if Redis is used only as cache:

```bash
redis-server --save "" --appendonly no
```

###  Redis Container Persistence Files — Definitions

| File / Folder                                              | Purpose / Definition                                                                                                                                                                                                                                                                                             |
| ---------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/data/dump.rdb`                                           | **RDB Snapshot File** — This is a binary snapshot of your Redis database at a certain point in time. Redis creates this file during a snapshot save (`SAVE` or `BGSAVE`). Deleting it removes the snapshot, but **does not stop Redis from running**.                                                            |
| `/data/appendonlydir/`                                     | **AOF Folder (Append Only File)** — This directory stores the append-only log of all write operations to Redis. It allows Redis to replay commands and restore the database after a restart. Deleting this folder removes all historical commands and persistent data. Redis will recreate it if AOF is enabled. |
| Other files inside `/data`                                 | Miscellaneous Redis persistence-related files (e.g., temporary files during save). Safe to leave untouched.                                                                                                                                                                                                      |
| Other container directories (e.g., `/usr`, `/etc`, `/var`) | System and Redis executable/config directories. **Critical for container operation**. Do **not delete**.                                                                                                                                                                                                         |
### Key Notes

    * Deleting dump.rdb or AOF files is safe for the container — only data is lost, Redis service itself keeps running.

    * Redis will recreate AOF folder if persistence is enabled on next startup.

    * This is a standard procedure when you want a full flush/reset in a production-safe Redis container where FLUSHALL is disabled.

## 14. References

* [Redis Persistence](https://redis.io/docs/management/persistence/)
* [Docker Redis Official Image](https://hub.docker.com/_/redis)



**Author:** Youssef Jehbali
**Date:** 26 February 2026
