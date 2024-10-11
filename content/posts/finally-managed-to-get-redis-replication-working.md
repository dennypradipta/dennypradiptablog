+++
title = 'Finally Managed to Get Redis Replication Working'
date = 2024-10-11T15:34:01+07:00
images = ['/finally-managed-to-get-redis-replication-working.webp']
+++

So I was trying to set up Redis replication with Redis Sentinel. I had this plan involving five servers:

-	10.10.10.1 - Master
-	10.10.10.2 - Slave
-	10.10.10.3, 10.10.10.4, 10.10.10.5 - Sentinels

Redis Sentinels are responsible for monitoring the health of the Redis master and slaves. If the master goes down, the sentinel will promote a slave to be the new master. If a slave goes down, the sentinel will promote a new slave to take its place.

Here are the Docker Compose files I used to set up the five servers:

1. Master (10.10.10.1)
```yaml
services:
  redis-master:
    image: bitnami/redis:6.2.6
    restart: unless-stopped
    environment:
      - REDIS_REPLICATION_MODE=master
      - REDIS_PASSWORD=nicely-done
    volumes:
      - ./data:/bitnami
    network_mode: host
```

2. Slave (10.10.10.2)
```yaml
services:
  redis-slave:
    image: bitnami/redis:6.2.6
    restart: unless-stopped
    environment:
      - REDIS_REPLICATION_MODE=slave
      - REDIS_REPLICA_IP=10.10.10.2
      - REDIS_MASTER_HOST=10.10.10.1
      - REDIS_MASTER_PORT_NUMBER=6379
      - REDIS_MASTER_PASSWORD=nicely-done
      - REDIS_PASSWORD=nicely-good
    volumes:
      - ./data:/bitnami
    network_mode: host
```

3. Sentinels (10.10.10.3, 10.10.10.4, 10.10.10.5)
```yaml
services:
  redis-sentinel:
    image: bitnami/redis-sentinel:6.2.6
    restart: unless-stopped
    network_mode: host
    environment:
      - REDIS_MASTER_HOST=10.10.10.1
      - REDIS_MASTER_PASSWORD=nicely-done
      - REDIS_SENTINEL_ANNOUNCE_IP=(10.10.10.3/10.10.10.4/10.10.10.5) # According to the Sentinel servers
      - REDIS_SENTINEL_ANNOUNCE_PORT=26379
      - REDIS_SENTINEL_DOWN_AFTER_MILLISECONDS=5000
      - REDIS_SENTINEL_FAILOVER_TIMEOUT=10000
    volumes:
      - ./data:/bitnami
```

When I run the Docker Compose files, everything works as expected. The master and slaves are able to communicate with each other, and the sentinels are able to monitor the health of the master and slaves. Take a look at the logs:

1. Master
```bash
$ docker compose exec -it redis-master redis-cli
127.0.0.1:6379> auth nicely-done
OK
127.0.0.1:6379> info replication
# Replication
role:master
connected_slaves:1
slave0:ip=10.10.10.2,port=6379,state=online,offset=66939,lag=1
master_failover_state:no-failover
master_replid:ac7df14482399c324613a0570a22ab91003dc091
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:67213
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:67213
127.0.0.1:6379> exit

$ docker compose logs
redis-master-1  | redis 04:04:33.36 INFO  ==> ** Starting Redis setup **
redis-master-1  | redis 04:04:33.39 INFO  ==> Initializing Redis
redis-master-1  | redis 04:04:33.41 INFO  ==> Setting Redis config file
redis-master-1  | redis 04:04:33.44 INFO  ==> Configuring replication mode
redis-master-1  | redis 04:04:33.47 INFO  ==> ** Redis setup finished! **
redis-master-1  |
redis-master-1  | redis 04:04:33.50 INFO  ==> ** Starting Redis **
redis-master-1  | 1:C 11 Oct 2024 04:04:33.513 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis-master-1  | 1:C 11 Oct 2024 04:04:33.513 # Redis version=6.2.6, bits=64, commit=00000000, modified=0, pid=1, just started
redis-master-1  | 1:C 11 Oct 2024 04:04:33.513 # Configuration loaded
redis-master-1  | 1:M 11 Oct 2024 04:04:33.514 * monotonic clock: POSIX clock_gettime
redis-master-1  | 1:M 11 Oct 2024 04:04:33.514 * Running mode=standalone, port=6379.
redis-master-1  | 1:M 11 Oct 2024 04:04:33.514 # Server initialized
redis-master-1  | 1:M 11 Oct 2024 04:04:33.514 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
redis-master-1  | 1:M 11 Oct 2024 04:04:33.515 * Ready to accept connections
redis-master-1  | 1:M 11 Oct 2024 04:04:33.905 * Replica 10.10.10.2:6379 asks for synchronization
redis-master-1  | 1:M 11 Oct 2024 04:04:33.905 * Partial resynchronization not accepted: Replication ID mismatch (Replica asked for 'fde979edfc5ededf1fd236e09932470573bd7d98', my replication IDs are '873211e79d2539472cbf037cd01eec05e9e10e36' and '0000000000000000000000000000000000000000')
redis-master-1  | 1:M 11 Oct 2024 04:04:33.905 * Replication backlog created, my new replication IDs are 'c19e2aa59d310f3c377aa34cdefe02722621a87b' and '0000000000000000000000000000000000000000'
redis-master-1  | 1:M 11 Oct 2024 04:04:33.905 * Starting BGSAVE for SYNC with target: disk
redis-master-1  | 1:M 11 Oct 2024 04:04:33.905 * Background saving started by pid 50
redis-master-1  | 50:C 11 Oct 2024 04:04:33.908 * DB saved on disk
redis-master-1  | 50:C 11 Oct 2024 04:04:33.909 * RDB: 0 MB of memory used by copy-on-write
redis-master-1  | 1:M 11 Oct 2024 04:04:33.917 * Background saving terminated with success
redis-master-1  | 1:M 11 Oct 2024 04:04:33.917 * Synchronization with replica 10.10.10.2:6379 succeeded
```

2. Slave
```bash
$ docker compose exec -it redis-replica redis-cli
127.0.0.1:6379> auth nicely-good
OK
127.0.0.1:6379> info replication
# Replication
role:slave
master_host:10.10.10.1
master_port:6379
master_link_status:up
master_last_io_seconds_ago:0
master_sync_in_progress:0
slave_read_repl_offset:76462
slave_repl_offset:76462
slave_priority:100
slave_read_only:1
replica_announced:1
connected_slaves:0
master_failover_state:no-failover
master_replid:ac7df14482399c324613a0570a22ab91003dc091
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:76462
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:3737
repl_backlog_histlen:72726
127.0.0.1:6379> exit

$ docker compose logs
redis-replica-1  | redis 04:19:52.07 INFO  ==> ** Starting Redis setup **
redis-replica-1  | redis 04:19:52.10 INFO  ==> Initializing Redis
redis-replica-1  | redis 04:19:52.12 INFO  ==> Setting Redis config file
redis-replica-1  | redis 04:19:52.14 INFO  ==> Configuring replication mode
redis-replica-1  | redis 04:19:52.19 INFO  ==> ** Redis setup finished! **
redis-replica-1  |
redis-replica-1  | redis 04:19:52.21 INFO  ==> ** Starting Redis **
redis-replica-1  | 1:C 11 Oct 2024 04:19:52.228 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis-replica-1  | 1:C 11 Oct 2024 04:19:52.228 # Redis version=6.2.6, bits=64, commit=00000000, modified=0, pid=1, just started
redis-replica-1  | 1:C 11 Oct 2024 04:19:52.228 # Configuration loaded
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.229 * monotonic clock: POSIX clock_gettime
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.230 * Running mode=standalone, port=6379.
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.230 # Server initialized
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.230 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * Reading RDB preamble from AOF file...
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * Loading RDB produced by version 6.2.6
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * RDB age 919 seconds
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * RDB memory usage when created 1.79 Mb
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * RDB has an AOF tail
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 # Done loading RDB, keys loaded: 0, keys expired: 0.
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * Reading the remaining AOF tail...
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * DB loaded from append only file: 0.000 seconds
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * Ready to accept connections
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * Connecting to MASTER 10.10.10.1:6379
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.231 * MASTER <-> REPLICA sync started
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.232 * Non blocking connect for SYNC fired the event.
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.232 * Master replied to PING, replication can continue...
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.232 * Partial resynchronization not possible (no cached master)
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.233 * Full resync from master: c19e2aa59d310f3c377aa34cdefe02722621a87b:1274
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.337 * MASTER <-> REPLICA sync: receiving 176 bytes from master to disk
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.337 * MASTER <-> REPLICA sync: Flushing old data
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.337 * MASTER <-> REPLICA sync: Loading DB in memory
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.340 * Loading RDB produced by version 6.2.6
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.340 * RDB age 0 seconds
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.340 * RDB memory usage when created 1.83 Mb
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.340 # Done loading RDB, keys loaded: 0, keys expired: 0.
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.340 * MASTER <-> REPLICA sync: Finished with success
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.340 * Background append only file rewriting started by pid 54
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.364 * AOF rewrite child asks to stop sending diffs.
redis-replica-1  | 54:C 11 Oct 2024 04:19:52.364 * Parent agreed to stop sending diffs. Finalizing AOF...
redis-replica-1  | 54:C 11 Oct 2024 04:19:52.364 * Concatenating 0.00 MB of AOF diff received from parent.
redis-replica-1  | 54:C 11 Oct 2024 04:19:52.364 * SYNC append only file rewrite performed
redis-replica-1  | 54:C 11 Oct 2024 04:19:52.365 * AOF rewrite: 0 MB of memory used by copy-on-write
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.433 * Background AOF rewrite terminated with success
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.433 * Residual parent diff successfully flushed to the rewritten AOF (0.00 MB)
redis-replica-1  | 1:S 11 Oct 2024 04:19:52.433 * Background AOF rewrite finished successfully
```

## The Problem

However, I noticed that the failover process is not working as expected. I tried to turn off the master, but it won't promote the slave to be the master. When I turn off the master, I observe the logs of the Sentinels:

1. Sentinel 1 (10.10.10.3)

```bash
redis-sentinel-1  | redis-sentinel 04:01:28.66
redis-sentinel-1  | redis-sentinel 04:01:28.67 INFO  ==> ** Starting Redis sentinel setup **
redis-sentinel-1  | redis-sentinel 04:01:28.70 INFO  ==> Initializing Redis Sentinel...
redis-sentinel-1  | redis-sentinel 04:01:28.70 INFO  ==> Persisted files detected, restoring...
redis-sentinel-1  | redis-sentinel 04:01:28.71 INFO  ==> ** Redis sentinel setup finished! **
redis-sentinel-1  | redis-sentinel 04:01:28.73 INFO  ==> ** Starting Redis Sentinel **
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:09.426 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:09.426 # Redis version=6.2.6, bits=64, commit=00000000, modified=0, pid=1, just started
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:09.426 # Configuration loaded
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:09.427 * monotonic clock: POSIX clock_gettime
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:09.428 * Running mode=sentinel, port=26379.
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:09.429 # Sentinel ID is cfc415ecf2999a460cfb68bb685b92865c77140e
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:09.429 # +monitor master mymaster 10.10.10.1 6379 quorum 2
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.447 # +sdown slave 10.10.10.2:6379 10.10.10.2 6379 @ mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.447 # +sdown sentinel 7f4d7e6f21bdc085b182f3c122809a0c1e0c2726 10.10.10.5 26379 @ mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.751 # -sdown sentinel 7f4d7e6f21bdc085b182f3c122809a0c1e0c2726 10.10.10.5 26379 @ mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.719 # +sdown master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.835 # +new-epoch 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.839 # +vote-for-leader 9385b6cd08afe315e2c012e335585867da43ef2d 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:25.845 # +odown master mymaster 10.10.10.1 6379 #quorum 3/2
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:25.845 # Next failover delay: I will not start a failover before Fri Oct 11 04:04:45 2024
```

2. Sentinel 2 (10.10.10.4)

```bash
redis-sentinel-1  | redis-sentinel 04:01:28.66
redis-sentinel-1  | redis-sentinel 04:01:28.67 INFO  ==> ** Starting Redis sentinel setup **
redis-sentinel-1  | redis-sentinel 04:01:28.70 INFO  ==> Initializing Redis Sentinel...
redis-sentinel-1  | redis-sentinel 04:01:28.70 INFO  ==> Persisted files detected, restoring...
redis-sentinel-1  | redis-sentinel 04:01:28.71 INFO  ==> ** Redis sentinel setup finished! **
redis-sentinel-1  | redis-sentinel 04:01:28.73 INFO  ==> ** Starting Redis Sentinel **
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:12.128 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:12.128 # Redis version=6.2.6, bits=64, commit=00000000, modified=0, pid=1, just started
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:12.128 # Configuration loaded
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:12.129 * monotonic clock: POSIX clock_gettime
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:12.130 * Running mode=sentinel, port=26379.
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:12.130 # Sentinel ID is 9385b6cd08afe315e2c012e335585867da43ef2d
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:12.130 # +monitor master mymaster 10.10.10.1 6379 quorum 2
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:17.172 # +sdown slave 10.10.10.2:6379 10.10.10.2 6379 @ mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.719 # +sdown master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.810 # +odown master mymaster 10.10.10.1 6379 #quorum 2/2
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.810 # +new-epoch 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.810 # +try-failover master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.814 # +vote-for-leader 9385b6cd08afe315e2c012e335585867da43ef2d 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.823 # cfc415ecf2999a460cfb68bb685b92865c77140e voted for 9385b6cd08afe315e2c012e335585867da43ef2d 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.823 # 7f4d7e6f21bdc085b182f3c122809a0c1e0c2726 voted for 9385b6cd08afe315e2c012e335585867da43ef2d 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.915 # +elected-leader master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.915 # +failover-state-select-slave master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.978 # -failover-abort-no-good-slave master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:25.031 # Next failover delay: I will not start a failover before Fri Oct 11 04:04:45 2024
```

3. Sentinel 3 (10.10.10.5)

```bash
redis-sentinel-1  | redis-sentinel 04:01:28.66
redis-sentinel-1  | redis-sentinel 04:01:28.67 INFO  ==> ** Starting Redis sentinel setup **
redis-sentinel-1  | redis-sentinel 04:01:28.70 INFO  ==> Initializing Redis Sentinel...
redis-sentinel-1  | redis-sentinel 04:01:28.70 INFO  ==> Persisted files detected, restoring...
redis-sentinel-1  | redis-sentinel 04:01:28.71 INFO  ==> ** Redis sentinel setup finished! **
redis-sentinel-1  | redis-sentinel 04:01:28.73 INFO  ==> ** Starting Redis Sentinel **
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.557 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.557 # Redis version=6.2.6, bits=64, commit=00000000, modified=0, pid=1, just started
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.557 # Configuration loaded
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.558 * monotonic clock: POSIX clock_gettime
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.559 * Running mode=sentinel, port=26379.
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.559 # Sentinel ID is 7f4d7e6f21bdc085b182f3c122809a0c1e0c2726
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:14.559 # +monitor master mymaster 10.10.10.1 6379 quorum 2
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:19.605 # +sdown slave 10.10.10.2:6379 10.10.10.2 6379 @ mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.726 # +sdown master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.813 # +new-epoch 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.817 # +vote-for-leader 9385b6cd08afe315e2c012e335585867da43ef2d 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.827 # +odown master mymaster 10.10.10.1 6379 #quorum 3/2
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.827 # Next failover delay: I will not start a failover before Fri Oct 11 04:04:45 2024
```

Pay attention to this logs from Sentinel 2:
```bash
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.810 # +try-failover master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.814 # +vote-for-leader 9385b6cd08afe315e2c012e335585867da43ef2d 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.823 # cfc415ecf2999a460cfb68bb685b92865c77140e voted for 9385b6cd08afe315e2c012e335585867da43ef2d 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.823 # 7f4d7e6f21bdc085b182f3c122809a0c1e0c2726 voted for 9385b6cd08afe315e2c012e335585867da43ef2d 38
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.915 # +elected-leader master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.915 # +failover-state-select-slave master mymaster 10.10.10.1 6379
redis-sentinel-1  | 1:X 11 Oct 2024 04:04:24.978 # -failover-abort-no-good-slave master mymaster 10.10.10.1 6379
```

It says that the failover was aborted because there was no good slave. The hell do you mean "no good slave"? I mean, the slave is up and running without any issues but it won't be promoted to be the master.

## The Solution

After digging StackOverflow, Gogole, Bitnami repos, and whatnot, I found the solution. Turns out, you need to have the same password for the master and the slave. I wasn't using the same password for the master and the slave (nicely-done and nicely-good), so when the sentinels tried to promote the slave to be the master, it failed because the passwords didn't match.

Here's a simplified flow for you to understand what would happen if the passwords is not the ssame:

1. Everything is running as usual
2. I turned off the master
3. "Oh damn, master is down. Reelect the primary now!" - Sentinel
4. Searching for a new master from a list of available replicas
5. "Found it. Let's use this replica. Oh, it needs auth, the master password is nicely-done according to our configuration" - Sentinel
6. Proceed to auth using nicely-done
7. Unauthorized because the replica password is nicely-good
8. "Oh damn, we are not allowed. Oh well, back to the original master" - Sentinel

So I changed the passwords for the master and the slave. Now here's a simplified flow if the passwords is the same:

1. Everything is running as usual
2. Turn off the master
3. "Oh damn, master is down. Reelect the primary now!" - Sentinel
4. Searching for a new master from a list of available replicas
5. "Found it. Let's use this replica. Oh, it needs auth, the master password is nicely-done according to our configuration" - Sentinel
6. Proceed to auth using nicely-done
7. Authorized

And that's it. The failover process is now working as expected. The logs of the sentinels tells me that the switch was successful:

```bash
redis-sentinel-1  | 9:X 11 Oct 2024 08:01:28.126 # +config-update-from sentinel f704a040c0f05d0e00e1697642f4bd8f09548432 10.10.10.3 26379 @ mymaster 10.10.10.1 6379
redis-sentinel-1  | 9:X 11 Oct 2024 08:01:28.126 # +switch-master mymaster 10.10.10.1 6379 10.10.10.2 6379
```

## Conclusion

This is the first time I have implemented a Redis replication. I spent two days trying to figure out why the failover wasn’t working, and guess what? It all boiled down to having different passwords for the master and the replica. Redis requires both to use the same password for the failover process to work properly. Once I fixed that, everything started working perfectly.

Lesson learned: **Always use the same password for your master and slave Redis instances if you want failover to happen smoothly.**

And remember, if all else fails, there’s always coffee. Lots of coffee.