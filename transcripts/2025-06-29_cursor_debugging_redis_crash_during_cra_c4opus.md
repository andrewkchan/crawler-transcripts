# Debugging redis crash during crawler run
_Exported on 6/29/2025 at 17:12:28 PDT from Cursor (1.0.0)_

---

**User**

Help me debug the root cause of my crawler crashing in an overnight run. The run began around 09:40. The crawler logs indicate that at some point around 15:57 redis restarted, causing all outstanding requests to fail, leading to a critical error:
```
2025-06-29 15:57:19 - CRITICAL - crawler_module.orchestrator [339299] - Orchestrator critical error: Redis is loading the dataset in memory
Traceback (most recent call last):
  File "/home/ubuntu/vibe-coding-experiments/crawler/crawler_module/orchestrator.py", line 1013, in run_crawl
    if await self.frontier.is_empty() and self.pages_crawled_count > 0: # Check if any crawling happened
       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/crawler_module/frontier.py", line 335, in is_empty
    count = await self.redis.llen('domains:queue')  # type: ignore[misc]
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/crawler_module/redis_shield.py", line 61, in llen
    return await self._shielded_execute(self._redis.llen(name))
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/crawler_module/redis_shield.py", line 31, in _shielded_execute
    return await asyncio.shield(coro)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/redis/asyncio/client.py", line 677, in execute_command
    return await conn.retry.call_with_retry(
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/redis/asyncio/retry.py", line 76, in call_with_retry
    raise error
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/redis/asyncio/retry.py", line 71, in call_with_retry
    return await do()
           ^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/redis/asyncio/client.py", line 652, in _send_command_parse_response
    return await self.parse_response(conn, command_name, **options)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/redis/asyncio/client.py", line 698, in parse_response
    response = await connection.read_response()
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/redis/asyncio/connection.py", line 594, in read_response
    response = await self._parser.read_response(
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/redis/_parsers/resp2.py", line 82, in read_response
    response = await self._read_response(disable_decoding=disable_decoding)
               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/redis/_parsers/resp2.py", line 102, in _read_response
    raise error
redis.exceptions.BusyLoadingError: Redis is loading the dataset in memory
```

The redis logs show that redis indeed restarted, but it's not obvious why to me:
```
...
1:M 29 Jun 2025 15:56:19.683 * Background saving terminated with success
1:M 29 Jun 2025 15:56:19.784 * Starting automatic rewriting of AOF on 105% growth
1:M 29 Jun 2025 15:56:19.794 * Creating AOF incr file appendonly.aof.503.incr.aof on background rewrite
1:M 29 Jun 2025 15:56:20.016 * Background append only file rewriting started by pid 69304
1:C 29 Jun 2025 15:57:14.663 # WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition. Being disabled, it can also cause failures without low memory condition, see https://github.com/jemalloc/jemalloc/issues/1328. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:C 29 Jun 2025 15:57:14.665 * oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 29 Jun 2025 15:57:14.665 * Redis version=8.0.2, bits=64, commit=00000000, modified=1, pid=1, just started
1:C 29 Jun 2025 15:57:14.665 * Configuration loaded
1:M 29 Jun 2025 15:57:14.665 * Increased maximum number of open files to 10032 (it was originally set to 1024).
1:M 29 Jun 2025 15:57:14.665 * monotonic clock: POSIX clock_gettime
1:M 29 Jun 2025 15:57:14.666 * Running mode=standalone, port=6379.
1:M 29 Jun 2025 15:57:14.670 * <bf> RedisBloom version 8.0.1 (Git=unknown)
1:M 29 Jun 2025 15:57:14.670 * <bf> Registering configuration options: [
1:M 29 Jun 2025 15:57:14.670 * <bf> 	{ bf-error-rate       :      0.01 }
1:M 29 Jun 2025 15:57:14.670 * <bf> 	{ bf-initial-size     :       100 }
1:M 29 Jun 2025 15:57:14.670 * <bf> 	{ bf-expansion-factor :         2 }
1:M 29 Jun 2025 15:57:14.670 * <bf> 	{ cf-bucket-size      :         2 }
1:M 29 Jun 2025 15:57:14.670 * <bf> 	{ cf-initial-size     :      1024 }
1:M 29 Jun 2025 15:57:14.670 * <bf> 	{ cf-max-iterations   :        20 }
1:M 29 Jun 2025 15:57:14.670 * <bf> 	{ cf-expansion-factor :         1 }
1:M 29 Jun 2025 15:57:14.670 * <bf> 	{ cf-max-expansions   :        32 }
1:M 29 Jun 2025 15:57:14.670 * <bf> ]
1:M 29 Jun 2025 15:57:14.670 * Module 'bf' loaded from /usr/local/lib/redis/modules//redisbloom.so
1:M 29 Jun 2025 15:57:14.693 * <search> Redis version found by RedisSearch : 8.0.2 - oss
1:M 29 Jun 2025 15:57:14.693 * <search> RediSearch version 8.0.1 (Git=5688fcc)
1:M 29 Jun 2025 15:57:14.694 * <search> Low level api version 1 initialized successfully
1:M 29 Jun 2025 15:57:14.694 * <search> gc: ON, prefix min length: 2, min word length to stem: 4, prefix max expansions: 200, query timeout (ms): 500, timeout policy: return, cursor read size: 1000, cursor max idle (ms): 300000, max doctable size: 1000000, max number of search results:  1000000, 
1:M 29 Jun 2025 15:57:14.695 * <search> Initialized thread pools!
1:M 29 Jun 2025 15:57:14.695 * <search> Disabled workers threadpool of size 0
1:M 29 Jun 2025 15:57:14.696 * <search> Subscribe to config changes
1:M 29 Jun 2025 15:57:14.696 * <search> Enabled role change notification
1:M 29 Jun 2025 15:57:14.698 * <search> Cluster configuration: AUTO partitions, type: 0, coordinator timeout: 0ms
1:M 29 Jun 2025 15:57:14.699 * <search> Register write commands
1:M 29 Jun 2025 15:57:14.699 * Module 'search' loaded from /usr/local/lib/redis/modules//redisearch.so
1:M 29 Jun 2025 15:57:14.704 * <timeseries> RedisTimeSeries version 80001, git_sha=577bfa8b5909e7ee572f0b651399be8303dc6641
1:M 29 Jun 2025 15:57:14.704 * <timeseries> Redis version found by RedisTimeSeries : 8.0.2 - oss
1:M 29 Jun 2025 15:57:14.704 * <timeseries> Registering configuration options: [
1:M 29 Jun 2025 15:57:14.704 * <timeseries> 	{ ts-compaction-policy   :              }
1:M 29 Jun 2025 15:57:14.704 * <timeseries> 	{ ts-num-threads         :            3 }
1:M 29 Jun 2025 15:57:14.704 * <timeseries> 	{ ts-retention-policy    :            0 }
1:M 29 Jun 2025 15:57:14.704 * <timeseries> 	{ ts-duplicate-policy    :        block }
1:M 29 Jun 2025 15:57:14.704 * <timeseries> 	{ ts-chunk-size-bytes    :         4096 }
1:M 29 Jun 2025 15:57:14.704 * <timeseries> 	{ ts-encoding            :   compressed }
1:M 29 Jun 2025 15:57:14.704 * <timeseries> 	{ ts-ignore-max-time-diff:            0 }
1:M 29 Jun 2025 15:57:14.704 * <timeseries> 	{ ts-ignore-max-val-diff :     0.000000 }
1:M 29 Jun 2025 15:57:14.704 * <timeseries> ]
1:M 29 Jun 2025 15:57:14.704 * <timeseries> Detected redis oss
1:M 29 Jun 2025 15:57:14.705 * Module 'timeseries' loaded from /usr/local/lib/redis/modules//redistimeseries.so
1:M 29 Jun 2025 15:57:14.717 * <ReJSON> Created new data type 'ReJSON-RL'
1:M 29 Jun 2025 15:57:14.719 * <ReJSON> version: 80001 git sha: unknown branch: unknown
1:M 29 Jun 2025 15:57:14.719 * <ReJSON> Exported RedisJSON_V1 API
1:M 29 Jun 2025 15:57:14.719 * <ReJSON> Exported RedisJSON_V2 API
1:M 29 Jun 2025 15:57:14.719 * <ReJSON> Exported RedisJSON_V3 API
1:M 29 Jun 2025 15:57:14.719 * <ReJSON> Exported RedisJSON_V4 API
1:M 29 Jun 2025 15:57:14.719 * <ReJSON> Exported RedisJSON_V5 API
1:M 29 Jun 2025 15:57:14.719 * <ReJSON> Enabled diskless replication
1:M 29 Jun 2025 15:57:14.720 * <ReJSON> Initialized shared string cache, thread safe: false.
1:M 29 Jun 2025 15:57:14.720 * Module 'ReJSON' loaded from /usr/local/lib/redis/modules//rejson.so
1:M 29 Jun 2025 15:57:14.720 * <search> Acquired RedisJSON_V5 API
1:M 29 Jun 2025 15:57:14.721 * Server initialized
1:M 29 Jun 2025 15:57:14.722 * <search> Loading event starts
1:M 29 Jun 2025 15:57:14.722 * <search> Enabled workers threadpool of size 4
1:M 29 Jun 2025 15:57:14.724 * Reading RDB base file on AOF loading...
1:M 29 Jun 2025 15:57:14.724 * Loading RDB produced by version 8.0.2
1:M 29 Jun 2025 15:57:14.724 * RDB age 1440 seconds
1:M 29 Jun 2025 15:57:14.724 * RDB memory usage when created 13691.50 Mb
1:M 29 Jun 2025 15:57:14.724 * RDB is base AOF
1:M 29 Jun 2025 15:58:06.853 * Done loading RDB, keys loaded: 13311854, keys expired: 0.
1:M 29 Jun 2025 15:58:06.853 * DB loaded from base file appendonly.aof.501.base.rdb: 52.131 seconds
1:M 29 Jun 2025 15:59:03.649 * DB loaded from incr file appendonly.aof.502.incr.aof: 56.796 seconds
1:M 29 Jun 2025 15:59:04.743 * DB loaded from incr file appendonly.aof.503.incr.aof: 1.094 seconds
1:M 29 Jun 2025 15:59:04.743 * <search> Disabled workers threadpool of size 4
1:M 29 Jun 2025 15:59:04.743 * <search> Loading event ends
1:M 29 Jun 2025 15:59:04.743 * DB loaded from append only file: 110.021 seconds
1:M 29 Jun 2025 15:59:04.743 * Opening AOF incr file appendonly.aof.503.incr.aof on server start
1:M 29 Jun 2025 15:59:04.743 * Ready to accept connections tcp
```

dmesg shows redis was oom-killed:
```
[Sun Jun 29 15:57:11 2025] node invoked oom-killer: gfp_mask=0x140cca(GFP_HIGHUSER_MOVABLE|__GFP_COMP), order=0, oom_score_adj=0
[Sun Jun 29 15:57:11 2025] CPU: 2 PID: 4282 Comm: node Not tainted 6.8.0-1030-aws #32-Ubuntu
[Sun Jun 29 15:57:11 2025] Hardware name: Amazon EC2 i7i.xlarge/, BIOS 1.0 10/16/2017
[Sun Jun 29 15:57:11 2025] Mem-Info:
[Sun Jun 29 15:57:11 2025] active_anon:4355954 inactive_anon:3278865 isolated_anon:0
                            active_file:170 inactive_file:655 isolated_file:0
                            unevictable:10531 dirty:0 writeback:33
                            slab_reclaimable:265505 slab_unreclaimable:57347
                            mapped:21906 shmem:20946 pagetables:38227
                            sec_pagetables:0 bounce:0
                            kernel_misc_reclaimable:0
                            free:51438 free_pcp:1007 free_cma:0
[Sun Jun 29 15:57:11 2025] Node 0 active_anon:17423816kB inactive_anon:13115460kB active_file:680kB inactive_file:2620kB unevictable:42124kB isolated(anon):0kB isolated(file):0kB mapped:87624kB dirty:0kB writeback:132kB shmem:83784kB shmem_thp:0kB shmem_pmdmapped:0kB anon_thp:0kB writeback_tmp:0kB kernel_stack:9024kB pagetables:152908kB sec_pagetables:0kB all_unreclaimable? no
[Sun Jun 29 15:57:11 2025] Node 0 DMA free:11264kB boost:0kB min:32kB low:44kB high:56kB reserved_highatomic:0KB active_anon:0kB inactive_anon:0kB active_file:0kB inactive_file:0kB unevictable:0kB writepending:0kB present:15996kB managed:15360kB mlocked:0kB bounce:0kB free_pcp:0kB local_pcp:0kB free_cma:0kB
[Sun Jun 29 15:57:11 2025] lowmem_reserve[]: 0 2975 31523 31523 31523
[Sun Jun 29 15:57:11 2025] Node 0 DMA32 free:120520kB boost:0kB min:6376kB low:9420kB high:12464kB reserved_highatomic:0KB active_anon:1599580kB inactive_anon:1287256kB active_file:0kB inactive_file:416kB unevictable:0kB writepending:0kB present:3125680kB managed:3060028kB mlocked:0kB bounce:0kB free_pcp:0kB local_pcp:0kB free_cma:0kB
[Sun Jun 29 15:57:11 2025] lowmem_reserve[]: 0 0 28547 28547 28547
[Sun Jun 29 15:57:11 2025] Node 0 Normal free:73968kB boost:129024kB min:190196kB low:219428kB high:248660kB reserved_highatomic:2048KB active_anon:15824180kB inactive_anon:11828260kB active_file:552kB inactive_file:2460kB unevictable:42124kB writepending:352kB present:29851648kB managed:29240228kB mlocked:27284kB bounce:0kB free_pcp:4028kB local_pcp:0kB free_cma:0kB
[Sun Jun 29 15:57:11 2025] lowmem_reserve[]: 0 0 0 0 0
[Sun Jun 29 15:57:11 2025] Node 0 DMA: 0*4kB 0*8kB 0*16kB 0*32kB 0*64kB 0*128kB 0*256kB 0*512kB 1*1024kB (U) 1*2048kB (M) 2*4096kB (M) = 11264kB
[Sun Jun 29 15:57:11 2025] Node 0 DMA32: 2328*4kB (UME) 1699*8kB (UME) 1343*16kB (UME) 793*32kB (UE) 377*64kB (UME) 134*128kB (UME) 35*256kB (UME) 1*512kB (M) 0*1024kB 0*2048kB 0*4096kB = 120520kB
[Sun Jun 29 15:57:11 2025] Node 0 Normal: 0*4kB 1*8kB (M) 1*16kB (M) 2334*32kB (UE) 0*64kB 0*128kB 0*256kB 0*512kB 0*1024kB 0*2048kB 0*4096kB = 74712kB
[Sun Jun 29 15:57:11 2025] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=1048576kB
[Sun Jun 29 15:57:11 2025] Node 0 hugepages_total=0 hugepages_free=0 hugepages_surp=0 hugepages_size=2048kB
[Sun Jun 29 15:57:11 2025] 27267 total pagecache pages
[Sun Jun 29 15:57:11 2025] 0 pages in swap cache
[Sun Jun 29 15:57:11 2025] Free swap  = 0kB
[Sun Jun 29 15:57:11 2025] Total swap = 0kB
[Sun Jun 29 15:57:11 2025] 8248331 pages RAM
[Sun Jun 29 15:57:11 2025] 0 pages HighMem/MovableOnly
[Sun Jun 29 15:57:11 2025] 169427 pages reserved
[Sun Jun 29 15:57:11 2025] 0 pages hwpoisoned
[Sun Jun 29 15:57:11 2025] oom-kill:constraint=CONSTRAINT_NONE,nodemask=(null),cpuset=user.slice,mems_allowed=0,global_oom,task_memcg=/system.slice/docker-97f0fa6b72ce802702bb17ba1f6cb7c5a0edbb18c59e08bf0bdd47ee0bf9195c.scope,task=redis-server,pid=60540,uid=999
[Sun Jun 29 15:57:11 2025] Out of memory: Killed process 60540 (redis-server) total-vm:17652600kB, anon-rss:14230956kB, file-rss:128kB, shmem-rss:0kB, UID:999 pgtables:31340kB oom_score_adj:0
```

I don't fully understand why this happened. Metrics (screenshot) show the system had 2GB available memory when the crawler crashed, and that redis was using around 14GB of memory. But the docker-compose config configures redis-server with `--maxmemory 24gb`. I guess the latter is at the level of the application running inside the container, though, and not necessarily the container itself. Thoughts?

See PLAN.MD for an overview of the crawler.

---

**Cursor**

Looking at the logs and metrics, I can see the root cause of your Redis crash. Let me examine the docker-compose configuration and explain what happened.

```plaintext
WARNING Memory overcommit must be enabled! Without it, a background save or replication may fail under low memory condition.
```

Now I understand the root cause of your crash. Let me examine the system's memory configuration to confirm my hypothesis.

## Root Cause Analysis

The crash was caused by Redis being OOM-killed during an AOF (Append Only File) rewrite operation. Here's what happened:

### The Problem Chain:

1. **Memory Overcommit is Disabled**: Your system has `vm.overcommit_memory = 0` (heuristic mode), which means Linux conservatively manages memory allocations.

2. **Redis Background Operations**: At 15:56:19, Redis started an AOF rewrite. This operation uses `fork()` to create a child process for the background work.

3. **Fork Memory Requirements**: With overcommit disabled, Linux requires enough free memory to potentially duplicate the entire parent process, even though copy-on-write (COW) means it won't actually use that much. With Redis using ~14GB, the fork needed to reserve another ~14GB.

4. **Insufficient Memory**: Your system had only ~2GB free memory, so the fork operation couldn't proceed. This triggered the OOM killer, which killed the Redis process.

### Why the Redis maxmemory Setting Didn't Help:

The `--maxmemory 24gb` setting is Redis's internal limit for data storage. It doesn't account for:
- Memory needed for fork operations during background saves/rewrites
- Redis process overhead beyond the data itself
- Container-level memory limits (which weren't set in your case)

### Solutions:

```shellscript
#!/bin/bash

# Solution 1: Enable memory overcommit (RECOMMENDED - immediate fix)
# This allows Redis background operations to work properly
echo "Current overcommit setting:"
sysctl vm.overcommit_memory

echo -e "\nTo fix immediately (temporary, until reboot):"
echo "sudo sysctl vm.overcommit_memory=1"

echo -e "\nTo fix permanently:"
echo "echo 'vm.overcommit_memory = 1' | sudo tee -a /etc/sysctl.conf"
echo "sudo sysctl -p"

# Solution 2: Adjust Redis memory limit to leave room for fork operations
echo -e "\n\nAlternative: Reduce Redis maxmemory to leave room for background operations"
echo "Edit docker-compose.yml and change:"
echo "  --maxmemory 24gb"
echo "to:"
echo "  --maxmemory 12gb"
echo "This leaves ~20GB for system + Redis fork operations"

# Solution 3: Add container memory limits
echo -e "\n\nAdditional safety: Add Docker container memory limits"
echo "In docker-compose.yml, under the redis service, add:"
echo "  deploy:"
echo "    resources:"
echo "      limits:"
echo "        memory: 16G"
echo "      reservations:"
echo "        memory: 14G"

# Solution 4: Disable Redis background saves if not needed
echo -e "\n\nIf you don't need persistence during crawling:"
echo "Remove these lines from docker-compose.yml:"
echo "  --save 900 1"
echo "  --save 300 10"
echo "  --save 60 10000"
echo "  --appendonly yes"
echo "Or use:"
echo "  --save \"\""
echo "  --appendonly no"
```

Now let me create a monitoring script to help you track Redis memory usage during future crawls:

```shellscript
#!/bin/bash

# Monitor Redis memory usage and system memory
echo "Monitoring Redis and system memory (Ctrl+C to stop)..."
echo "Time | Redis RSS | Redis Dataset | System Free | System Available"
echo "-----+------------+---------------+-------------+-----------------"

while true; do
    # Get Redis memory info
    REDIS_INFO=$(docker exec crawler-redis redis-cli INFO memory 2>/dev/null)
    if [ $? -eq 0 ]; then
        REDIS_RSS=$(echo "$REDIS_INFO" | grep "used_memory_rss_human:" | cut -d: -f2 | tr -d '\r')
        REDIS_DATASET=$(echo "$REDIS_INFO" | grep "used_memory_dataset_perc:" | cut -d: -f2 | tr -d '\r' | tr -d '%')
        REDIS_USED=$(echo "$REDIS_INFO" | grep "used_memory_human:" | cut -d: -f2 | tr -d '\r')
    else
        REDIS_RSS="N/A"
        REDIS_DATASET="N/A"
        REDIS_USED="N/A"
    fi
    
    # Get system memory
    MEM_INFO=$(free -h | grep "^Mem:")
    SYS_FREE=$(echo "$MEM_INFO" | awk '{print $4}')
    SYS_AVAILABLE=$(echo "$MEM_INFO" | awk '{print $7}')
    
    # Print current stats
    printf "%s | %-10s | %-13s | %-11s | %-15s\n" \
        "$(date '+%H:%M:%S')" \
        "$REDIS_RSS" \
        "${REDIS_USED} (${REDIS_DATASET}%)" \
        "$SYS_FREE" \
        "$SYS_AVAILABLE"
    
    # Check for critical memory condition
    AVAILABLE_GB=$(free -g | grep "^Mem:" | awk '{print $7}')
    if [ "$AVAILABLE_GB" -lt 3 ]; then
        echo "WARNING: Available memory below 3GB! Redis background operations may fail!"
    fi
    
    sleep 5
done
```

---

**User**

can you add these metrics to the orchestrator periodic logging in _log_metrics instead (see "Basic Redis metrics") if there are any that are not already there?

---

**Cursor**

Looking at the existing Redis metrics in `_log_metrics`, I can see that some useful metrics from the monitoring script aren't being logged. Let me add those additional Redis memory metrics to provide better visibility:

```python
            # Redis metrics
            redis_info = await self.redis_client.info()
            redis_memory = await self.redis_client.info('memory')
            redis_stats = await self.redis_client.info('stats')
            
            # Basic Redis metrics
            redis_memory_usage_gauge.set(redis_memory.get('used_memory', 0))
            redis_connected_clients_gauge.set(redis_info.get('connected_clients', 0))
            
            # Redis ops/sec
            instantaneous_ops = redis_stats.get('instantaneous_ops_per_sec', 0)
            redis_ops_per_sec_gauge.set(instantaneous_ops)
            
            # Calculate hit rate
            if self.last_redis_stats:
                time_diff = current_time - self.last_redis_stats_time
                if time_diff > 0:
                    hits_diff = redis_stats.get('keyspace_hits', 0) - self.last_redis_stats.get('keyspace_hits', 0)
                    misses_diff = redis_stats.get('keyspace_misses', 0) - self.last_redis_stats.get('keyspace_misses', 0)
                    total_diff = hits_diff + misses_diff
                    if total_diff > 0:
                        hit_rate = (hits_diff / total_diff) * 100
                        redis_hit_rate_gauge.set(hit_rate)
                        logger.info(f"[Metrics] Redis: Ops/sec={instantaneous_ops}, Memory={redis_memory.get('used_memory_human', 'N/A')}, Hit Rate={hit_rate:.1f}%")
                    else:
                        logger.info(f"[Metrics] Redis: Ops/sec={instantaneous_ops}, Memory={redis_memory.get('used_memory_human', 'N/A')}")
            else:
                logger.info(f"[Metrics] Redis: Ops/sec={instantaneous_ops}, Memory={redis_memory.get('used_memory_human', 'N/A')}")

->

            # Redis metrics
            redis_info = await self.redis_client.info()
            redis_memory = await self.redis_client.info('memory')
            redis_stats = await self.redis_client.info('stats')
            
            # Basic Redis metrics
            redis_memory_usage_gauge.set(redis_memory.get('used_memory', 0))
            redis_connected_clients_gauge.set(redis_info.get('connected_clients', 0))
            
            # Redis ops/sec
            instantaneous_ops = redis_stats.get('instantaneous_ops_per_sec', 0)
            redis_ops_per_sec_gauge.set(instantaneous_ops)
            
            # Additional Redis memory metrics
            redis_rss = redis_memory.get('used_memory_rss_human', 'N/A')
            redis_dataset_perc = redis_memory.get('used_memory_dataset_perc', 'N/A')
            redis_used = redis_memory.get('used_memory_human', 'N/A')
            redis_max = redis_memory.get('maxmemory_human', 'N/A')
            redis_evicted_keys = redis_stats.get('evicted_keys', 0)
            
            # Calculate hit rate
            if self.last_redis_stats:
                time_diff = current_time - self.last_redis_stats_time
                if time_diff > 0:
                    hits_diff = redis_stats.get('keyspace_hits', 0) - self.last_redis_stats.get('keyspace_hits', 0)
                    misses_diff = redis_stats.get('keyspace_misses', 0) - self.last_redis_stats.get('keyspace_misses', 0)
                    total_diff = hits_diff + misses_diff
                    if total_diff > 0:
                        hit_rate = (hits_diff / total_diff) * 100
                        redis_hit_rate_gauge.set(hit_rate)
                        logger.info(f"[Metrics] Redis: Ops/sec={instantaneous_ops}, Memory={redis_used}/{redis_max} (RSS: {redis_rss}, Dataset: {redis_dataset_perc}), Hit Rate={hit_rate:.1f}%, Evicted={redis_evicted_keys}")
                    else:
                        logger.info(f"[Metrics] Redis: Ops/sec={instantaneous_ops}, Memory={redis_used}/{redis_max} (RSS: {redis_rss}, Dataset: {redis_dataset_perc}), Evicted={redis_evicted_keys}")
            else:
                logger.info(f"[Metrics] Redis: Ops/sec={instantaneous_ops}, Memory={redis_used}/{redis_max} (RSS: {redis_rss}, Dataset: {redis_dataset_perc}), Evicted={redis_evicted_keys}")
```

Good! I've added the additional Redis memory metrics to the periodic logging. Now let me also add a warning when Redis memory usage is getting close to its configured limit, which would help prevent future OOM situations:

```python
            # Additional Redis memory metrics
            redis_rss = redis_memory.get('used_memory_rss_human', 'N/A')
            redis_dataset_perc = redis_memory.get('used_memory_dataset_perc', 'N/A')
            redis_used = redis_memory.get('used_memory_human', 'N/A')
            redis_max = redis_memory.get('maxmemory_human', 'N/A')
            redis_evicted_keys = redis_stats.get('evicted_keys', 0)

->

            # Additional Redis memory metrics
            redis_rss = redis_memory.get('used_memory_rss_human', 'N/A')
            redis_dataset_perc = redis_memory.get('used_memory_dataset_perc', 'N/A')
            redis_used = redis_memory.get('used_memory_human', 'N/A')
            redis_max = redis_memory.get('maxmemory_human', 'N/A')
            redis_evicted_keys = redis_stats.get('evicted_keys', 0)
            
            # Check Redis memory usage percentage and warn if high
            used_memory_bytes = redis_memory.get('used_memory', 0)
            max_memory_bytes = redis_memory.get('maxmemory', 0)
            if max_memory_bytes > 0:
                memory_usage_percent = (used_memory_bytes / max_memory_bytes) * 100
                if memory_usage_percent > 90:
                    logger.warning(f"[Metrics] REDIS MEMORY WARNING: Using {memory_usage_percent:.1f}% of maxmemory ({redis_used}/{redis_max})")
                elif memory_usage_percent > 80:
                    logger.info(f"[Metrics] Redis memory usage at {memory_usage_percent:.1f}% of maxmemory")
```

Let me also add a warning for low system memory, which is critical for Redis background operations:

```python
            # Memory metrics
            mem = psutil.virtual_memory()
            memory_free_gauge.set(mem.free)
            memory_available_gauge.set(mem.available)
            logger.info(f"[Metrics] Memory: Free={mem.free/1024/1024/1024:.1f}GB, Available={mem.available/1024/1024/1024:.1f}GB, Used={mem.percent:.1f}%")

->

            # Memory metrics
            mem = psutil.virtual_memory()
            memory_free_gauge.set(mem.free)
            memory_available_gauge.set(mem.available)
            logger.info(f"[Metrics] Memory: Free={mem.free/1024/1024/1024:.1f}GB, Available={mem.available/1024/1024/1024:.1f}GB, Used={mem.percent:.1f}%")
            
            # Warn if system memory is low (important for Redis fork operations)
            available_gb = mem.available / 1024 / 1024 / 1024
            if available_gb < 2.0:
                logger.warning(f"[Metrics] SYSTEM MEMORY WARNING: Only {available_gb:.1f}GB available - Redis background operations may fail!")
            elif available_gb < 4.0:
                logger.info(f"[Metrics] System memory getting low: {available_gb:.1f}GB available")
```

Perfect! I've added the following enhancements to the orchestrator's `_log_metrics` method:

1. **Additional Redis memory metrics**:
   - RSS memory usage (physical memory used by Redis)
   - Dataset percentage (how much of memory is actual data vs overhead)
   - Maximum memory configuration
   - Evicted keys count

2. **Redis memory usage warnings**:
   - Warns when Redis uses >90% of its configured maxmemory
   - Info message when Redis uses >80% of maxmemory

3. **System memory warnings**:
   - Critical warning when available system memory drops below 2GB
   - Info message when available memory is below 4GB

These metrics will help you monitor and prevent future OOM situations. The system memory warning is particularly important because Redis needs enough free memory to fork during background operations (AOF rewrites, RDB saves).

Now let me clean up the temporary files:

I've kept the `monitor-redis-memory.sh` script as it could be useful for real-time monitoring during future crawls. You can run it in a separate terminal with:

```bash
cd crawler
./monitor-redis-memory.sh
```

The enhanced metrics in the orchestrator will now provide better visibility into both Redis and system memory usage, helping you catch potential OOM situations before they cause crashes. The key indicators to watch for are:

1. **System available memory < 2GB** - Critical for Redis fork operations
2. **Redis memory usage > 80%** of configured maxmemory
3. **Redis RSS significantly higher than used memory** - indicates memory fragmentation

Based on your crash analysis, the immediate fix is still to enable memory overcommit:
```bash
sudo sysctl vm.overcommit_memory=1
```

This will allow Redis background operations to succeed even when available memory is low.

```shellscript
cd crawler
./monitor-redis-memory.sh
```

```shellscript
sudo sysctl vm.overcommit_memory=1
```

