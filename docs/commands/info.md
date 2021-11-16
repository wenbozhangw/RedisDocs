## INFO [section]

    起始版本：1.0.0

[INFO](info.md) 命令以一种易于理解和阅读的格式，返回 Redis 服务器的各种信息和统计数值。

通过给定可选的参数 `section`，可以让命令只返回某一部分信息：
- `server`：Redis 服务器的一般信息
- `clients`：客户端的连接部分
- `memory`：内存消耗相关信息
- `persistence`：RDB 和 AOF 相关信息
- `stats`：一般统计
- `replication`：Master/replica 复制信息
- `cpu`：统计 CPU 的消耗
- `commandstats`：Redis 命令统计
- `cluster`：Redis Cluster 部分
- `modules`：Modules 部分
- `keyspace`：数据库的相关统计
- `modules`：模块相关部分
- `errorstats`：Redis 错误统计

也可以采用以下值：
- `all`：返回所有信息（不包括模块生成的）
- `default`：只返回默认设置的信息
- `everything`：包括 `all` 和 `modules`

如果没有使用任何参数时，默认为 `default`。

---

### Return Value

[Bulk string reply](../topics/protocol.md#resp-bulk-strings)：文本行的集合。

行可以包含节点名称（以 # 字符开头）或属性。所有属性都采用 `field:value` 的形式，以 `\r\n` 结尾。

```
redis> INFO
# Server
redis_version:6.2.6
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:e72639486d2ec1a2
redis_mode:standalone
os:Linux 5.4.0-1030-aws x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:c11-builtin
gcc_version:9.3.0
process_id:3307900
process_supervised:no
run_id:6c93018a2a9facdb95803b70c7508e5fc63a5a97
tcp_port:6379
server_time_usec:1634195991665709
uptime_in_seconds:849211
uptime_in_days:9
hz:10
configured_hz:10
lru_clock:6806039
executable:/usr/local/bin/redis-server
config_file:/etc/redis/redis.conf
io_threads_active:0

# Clients
connected_clients:2
cluster_connections:0
maxclients:10000
client_recent_max_input_buffer:40
client_recent_max_output_buffer:0
blocked_clients:0
tracking_clients:0
clients_in_timeout_table:0

# Memory
used_memory:124807640
used_memory_human:119.03M
used_memory_rss:133464064
used_memory_rss_human:127.28M
used_memory_peak:124875376
used_memory_peak_human:119.09M
used_memory_peak_perc:99.95%
used_memory_overhead:33577176
used_memory_startup:809912
used_memory_dataset:91230464
used_memory_dataset_perc:73.57%
allocator_allocated:124806944
allocator_active:125829120
allocator_resident:133128192
total_system_memory:16596955136
total_system_memory_human:15.46G
used_memory_lua:37888
used_memory_lua_human:37.00K
used_memory_scripts:0
used_memory_scripts_human:0B
number_of_cached_scripts:0
maxmemory:200000000
maxmemory_human:190.73M
maxmemory_policy:allkeys-lru
allocator_frag_ratio:1.01
allocator_frag_bytes:1022176
allocator_rss_ratio:1.06
allocator_rss_bytes:7299072
rss_overhead_ratio:1.00
rss_overhead_bytes:335872
mem_fragmentation_ratio:1.07
mem_fragmentation_bytes:8740840
mem_not_counted_for_evict:0
mem_replication_backlog:0
mem_clients_slaves:0
mem_clients_normal:41024
mem_aof_buffer:0
mem_allocator:jemalloc-5.1.0
active_defrag_running:0
lazyfree_pending_objects:0
lazyfreed_objects:0

# Persistence
loading:0
current_cow_size:0
current_cow_size_age:0
current_fork_perc:0.00
current_save_keys_processed:0
current_save_keys_total:0
rdb_changes_since_last_save:3056714
rdb_bgsave_in_progress:0
rdb_last_save_time:1633346780
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
rdb_last_cow_size:0
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_last_cow_size:0
module_fork_in_progress:0
module_fork_last_cow_size:0

# Stats
total_connections_received:56703
total_commands_processed:7434855
instantaneous_ops_per_sec:8
total_net_input_bytes:618188861
total_net_output_bytes:111029308
instantaneous_input_kbps:0.73
instantaneous_output_kbps:0.10
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:20644
expired_stale_perc:0.11
expired_time_cap_reached_count:0
expire_cycle_cpu_milliseconds:54340
evicted_keys:0
keyspace_hits:1727491
keyspace_misses:773940
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0
total_forks:0
migrate_cached_sockets:0
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0
tracking_total_keys:0
tracking_total_items:0
tracking_total_prefixes:0
unexpected_error_replies:0
total_error_replies:22980
dump_payload_sanitizations:0
total_reads_processed:7475755
total_writes_processed:7475748
io_threaded_reads_processed:0
io_threaded_writes_processed:0

# Replication
role:master
connected_slaves:0
master_failover_state:no-failover
master_replid:fbe919e4b02713bbecaef7edc1b29c0ad1a6913a
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0

# CPU
used_cpu_sys:829.553046
used_cpu_user:2821.449350
used_cpu_sys_children:0.000000
used_cpu_user_children:0.000000
used_cpu_sys_main_thread:820.924638
used_cpu_user_main_thread:2812.245259

# Modules

# Errorstats
errorstat_ERR:count=22225
errorstat_WRONGTYPE:count=755

# Cluster
cluster_enabled:0

# Keyspace
db0:keys=607780,expires=760,avg_ttl=31253190376853
```

---

### Notes

请注意不同 Redis 版本会添加后者删除一些字段。一个健壮的客户端应该解析该命令的结果时，应该跳过未知字段，并且优雅地处理缺少的字段。

一下是 Redis 大于等于 2.4 版本的字段说明。

下面是所有 **server** 相关的信息：

- `redis_version`：Redis 服务器版本。
- `redis_git_sha1`：Git SHA1。
- `redis_git_dirty`：Git dirty flag。
- `redis_build_id`：构建 ID。
- `redis_mode`：服务器模式（"standalone","sentinel" 或 "cluster"）。
- `os`：Redis 服务器的宿主操作系统。
- `arch_bits`：架构（32位 或 64 位）。
- `multiplexing_api`：Redis 所使用的事件处理机制。
- `atomicvar_api`：Redis 使用的 Atomicvar API。
- `gcc_version`：编译 Redis 时所使用的 GCC 版本。
- `process_id`：服务器进程的 PID。
- `process_supervised`：Supervised system（"upstart"、"systemd"、"unknown" 或 "no"）。
- `run_id`：Redis 服务器的随机标识符（用于 Sentinel 和 Cluster）。
- `tcp_port`：TCP/IP 监听端口。
- `server_time_in_usec`：具有微秒简单过度的 Epoch-based system time。
- `uptime_in_seconds`：自 Redis 服务器启动以来，经过的秒数。
- `uptime_in_day`：自 Redis 服务器启动以来，经过的天数。
- `hz`：服务器的当前频率设置。
- `configured_hz`：服务器的配置频率设置。
- `lru_clock`：以分钟为单位进行自增的始终，用于 LRU 管理。
- `executable`：服务器的可执行文件路径。
- `config_file`：配置文件的路径。

下面是所有 **clients** 相关的信息：
- `connected_clients`：已经连接客户端的数量（不包括通过 replicas 服务器连接的客户端）。
- `cluster_connections`：集群总线使用的 socket 数量的近似值。
- `maxclients`：`maxclients`配置指令的值。这是 `connected_clients`、`connected_slaves` 和 `cluster_connections` 总和的上限。
- `client_longest_output_list`：当前客户端连接中最长地输出列表。
- `client_biggest_input_buf`：当前连接的客户端中，最大的输入缓冲区。
- `blocked_clients`：正在等待阻塞命令（[BLPOP](blpop.md)、[BRPOP](brpop.md)、[BRPOPLPUSH](brpoplpush.md)、[BLMOVE](blmove.md)、[BZPOPMIN](bzpopmin.md)、[BZPOPMAX](bzpopmax.md)）的客户端的数量。
- `tracking_clients`：被跟踪的客户端数量（[CLIENT TRACKING](client-tracking.md)）。
- `clients_in_timeout_table`：客户端超时表中的客户端数量。
- `io_threads_active`：指示 I/O 线程是否处于活动状态的标志。

下面是所有 **memory** 相关信息：
- `used_memory`：由 Redis 分配器分配的内存总量，以字节 (byte) 为单位（标准 **libc, jemalloc** 或 alternative allocator，例如 [tcmalloc](http://code.google.com/p/google-perftools/) ）。
- `used_memory_human`：以人类可读的格式返回 Redis 分配的内存总量。
- `used_memory_rss`：从操作系统的角度，返回 Redis 已分配的内存总量（俗称常驻集大小）。这个值和 top、ps 等命令的输出一致。
- `used_memory_rss_human`：以人类可读的格式返回 `used_memory_rss` 值。
- `used_memory_peak`： Redis 的内存消耗峰值（以字节为单位）。
- `used_memory_peak_human`：以人类可读的格式返回 `used_memory_peak` 值。
- `used_memory_peak_perc`：`used_memory_peak` 占 `used_memory` 的百分比。
- `used_memory_overhead`：服务器为管理其内部数据结构而分配的所有开销的总和（以字节为单位）。
- `used_memory_startup`：Redis 在启动时消耗的初始内存大小（以字节为单位）。
- `used_memory_dataset`：以字节为单位的数据集大小（`used_memory` 减去 `used_memory_startup`）。
- `total_system_memory`：Redis 主机具有的内存总量。
- `total_system_memory_human`：以人类可读的格式返回 `total_system_memory` 值。
- `used_memory_lua`：Lua 引擎所使用的内存大小（以字节为单位）。
- `used_memory_lua_human`：以人类可读的格式返回 `used_memory_lua` 值。
- `maxmemory`：maxmemory 配置指令的值。
- `maxmemory_human`：以人类可读的方式返回 `maxmemory` 的值。
- `maxmemory_policy`：`maxmemory-policy` 配置指令的值。
- `mem_fragmentation_ratio`：`used_memory_rss` 和 `used_memory` 之间的比率。请注意，这不仅包括碎片(fragmentation)，还包括其他进程开销（请阅读 **allocator_** 指标），以及代码，共享课，堆栈开销等。
- `mem_fragmentation_bytes`：`used_memory_rss` 和 `used_memory` 之间的差值。请注意，当总碎片字节数较低（几兆字节）时，高比率（例如 1.5 及以上）并不表示存在问题。
- `allocator_frag_ratio`：`allocator_active` 和 `allocator_allocated` 的比率。这是真正的（外部）碎片指标（不是 `meme_fragmentation_ratio`）。
- `allocator_frag_bytes`：`allocator_active` 和 `allocator_allocated` 之间的增量。请参阅有关 `mem_fragmentation_bytes` 的说明。
- `allocator_rss_ratio`：`allocator_resident` 和 `allocator_active` 之间的比率。这通常表示分配器可以并且可能很快将释放回操作系统的 page。
- `allocator_rss_bytes`：`allocator_resident` 和 `allocator_active` 之间的差值。
- `rss_overhead_ratio`：`used_memory_rss` （进程 RSS）和 `allocator_resident` 之间的比率。这包括与分配器或堆无关的 RSS 开销。
- `rss_overhead_bytes`：`used_memory_rss`（进程 RSS）和 `allocator_resident` 之间的差值。
- `allocator_allocated`：从分配器分配的总字节数，包括内部碎片(internal-fragmentation)。通常与 `used_memory` 相同。
- `allocator_active`：分配器活动 page 中的总字节数，这包括外部碎片(external-fragmentation)。
- `allocator_resident`：分配器中的总字节数（RSS），这包括可以释放到操作系统的 page（通过 [MEMORY PURGE](memory-purge.md)，或者只是等待）。
- `mem_allocator`：内存分配器(memory allocator)，在编译时选择。
- `active_defrag_running`：当启用 `activedefrag` 时，这表示碎片整理当前是否处于活动状态，以及它打算使用的 CPU 百分比。
- `lazyfree_pending_objects`：等待被释放的对象数（由于调用 [UNLINK](unlink.md) 或带有 **ASYNC** 选项的 [FLUSHDB](flushdb.md) 和 [FLUSHALL](flushall.md) ）。

理想情况下，`used_memory_rss` 值应该仅略高于 `used_memory`。当 **rss >> used** ，且两者的值相差较大时，表示存在（外部的）内存碎片，可以通过检查 `allocator_frag_ratio`、`allocator_frag_bytes` 来评估。当 **used >> rss** 时，这意味着部分 Redis 内存已经被操作系统换出到交换空间了：在这种情况下，操作可能会产生明显的延迟。

由于 Redis 无法控制其分配的内存如何映射到内存页，因此 `used_memory_rss` 高通常是内存使用量激增的结果。

Redis 在释放内存时，会将内存还给分配器，分配器可能会，也可能不会将内存还给系统。操作系统报告的 `used_memory` 值和内存消耗之间可能存在差异。可能是因为内存已经被 Redis 使用和释放了，没有还给系统。`used_memory_peak` 值通常用于检查这一点。

可以通过参考 [MEMORY STATS](memory-stats.md) 命令和 [MEMORY DOCTOR](memory-doctor.md) 获得有关服务器内存的其他内省信息(introspective information)。

下面是所有 **persistence** 相关信息：
- `loading`：指示 dump file 的加载是否正在进行的标志。
- `current_cow_peak`：child fork 运行时 copy-on-write 内存的峰值大小（以字节为单位）。
- `current_cow_size`：child fork 运行时 copy-on-write 内存的大小（以字节为单位）。
- `current_fork_perc`：当前 fork 进程的进度百分比。对于 AOF 和 RDB fork，它是 `current_save_keys_processed` 占 `current_save_keys_total` 的百分比。
- `current_save_keys_processed`：当前保存操作处理的 key 数量。
- `current_save_keys_total`：当前保存操作开始时的 key 数量。
- `rdb_changes_since_last_save`：自上次 dump 以来的更改次数。
- `rdb_bgsave_in_progress`：指示 RDB 保存正在进行的标志。
- `rdb_last_save_time`：上次成功保存 RDB 的 epoch-based 时间戳。
- `rdb_last_bgsave_status`：上次 RDB 保存操作的状态。
- `rdb_last_bgsave_time_sec`：正在进行的 RDB 保存操作的持续时间（如果有）。
- `rdb_last_cow_size`：上次 RDB 保存操作期间 copy-on-write 内存的大小（以字节为单位）。
- `aof_enabled`：表示 AOF logging 已激活的标志。
- `aof_rewrite_in_progress`：表示 AOF 长些操作正在进行的标志。
- `aof_rewrite_scheduled`：表示一旦进行中的 RDB 保存操作完成，就会安排进行 AOF 重写操作的标志。
- `aof_last_rewrite_time_sec`：上次 AOF 重写操作的持续时间（以秒为单位）。
- `aof_current_rewrite_time_sec`：正在进行的 AOF 重写操作的持续时间（以秒为单位）。
- `aof_last_bgrewrite_status`：上次 AOF 重写操作的状态。
- `aof_last_write_status`：对 AOF 的最后一次写操作的状态。
- `aof_last_cow_size`：上次 AOF 重写操作期间 copy-on-write 内存的大小（以字节为单位）。
- `module_fork_in_progress`：指示模块 fork 正在进行的标志。
- `module_fork_last_cow_size`：上次模块 fork 操作期间的 copy-on-write 内存的大小（以字节为单位）。

`rdb_changes_since_last_save` 指的是从上次调用 [SAVE](save.md) 或者 [BGSAVE](bgsave.md) 依赖，在数据集中产生某种变化的操作的数量。

如果启用了 AOF，则会添加以下这些额外的字段：
- `aof_current_size`：当前的 AOF 文件大小。
- `aof_base_size`：上次启动或重写时，AOF文件大小。
- `aof_pending_rewrite`：指示 AOF 重写操作是否会在当前 RDB 保存操作完成后立即执行的标志。
- `aof_buffer_length`：AOF 缓冲区大小。
- `aof_rewrite_buffer_length`：AOF 重写缓冲区大小。
- `aof_pending_bio_fsync`：在后台 IO 队列中等待 fsync 处理的任务数。
- `aof_delayed_fsync`：延迟 fsync 计数器。

如果正在执行加载操作，将会添加这些额外的字段：
- `loading_start_time`：加载操作的开始时间戳，Epoch-based timestamp。
- `loading_total_bytes`：文件总大小。
- `loading_rdb_used_mem`：创建文件时生成 RDB 文件的服务器的内存使用情况。
- `loading_loaded_bytes`：已经加载的字节数。
- `loading_loaded_perc`：已经加载的百分比。
- `loading_eta_seconds`：预计加载完成所需的剩余秒数。

下面是所有 **stats** 相关信息：
- `total_connections_received`：服务器接受的连接总数。
- `total_commands_processed`：服务器处理的命令总数。
- `instantaneous_ops_per_sec`：每秒处理的命令数。
- `total_net_input_bytes`：从网络读取的总字节数。
- `total_net_output_bytes`：写入网络的总字节数。
- `instantaneous_input_kbps`：每秒网络读取速率，以 KB/sec 为单位。
- `instantaneous_output_kbps`：每秒写入网络速率，以 KB/sec 为单位。
- `rejected_connections`：由于 `maxclients` 限制而拒绝的连接数。
- `sync_full`：与副本完全重新同步(full resyncs)的次数。
- `sync_partial_ok`：接收的部分重新同步(partial resync)请求的数量。
- `sync_partial_err`：被拒绝的部分重新同步请求的数量。
- `expired_keys`：key 到期事件的总数。
- `expired_stale_perc`：可能已过期的 key 百分比。
- `expired_time_cap_reached_count`：有效到期周期(active expiry cycles)提前停止的次数。
- `expire_cycle_cpu_milliseconds`：在有效到期周期上花费的累积时间。
- `evicted_keys`：由于 `maxmemory` 限制而导致被逐出的 key 的数量。
- `total_eviction_exceeded_time`：自服务器启动以来 `used_memory` 大于 `maxmemory` 的总时间，以毫秒为单位。
- `current_eviction_exceeded_time`：自 `used_memory` 上次超过 `maxmemory` 以来经过的时间，以毫秒为单位。
- `keyspace_hits`：在主字典(main dictionary)中成功查找 key 的次数。
- `keyspace_minsses`：在主字典(main dictionary)中查找 key 失败的次数。
- `pubsub_channels`：拥有客户端订阅和全局 pub/sub channel 数量。
- `pubsub_patterns`：拥有客户端订阅和全局 pub/sub pattern 数量。
- `lastest_fork_usec`：最新 fork 操作的持续时间，以微秒为单位。
- `total_forks`：自服务器启动以来的 fork 操作总数。
- `migrate_cached_sockets`：为 [MIGRATE](migrate.md) 目的打开的 socket 数量。
- `slave_expires_tracked_keys`：为到期目的跟踪的 key 数量（仅适用于可写副本）。
- `active_defrag_hits`：激活碎片整理过程执行的值重新分配的数量。
- `active_defrag_misses`：激活碎片整理过程启动的终止值重新分配的数量。
- `active_defrag_key_hits`：激活碎片整理的 key 数量。
- `active_defrag_key_misses`：激活碎片整理过程跳过的 key 数量。
- `total_active_defrag_time`：内存碎片超过限制的总时间，以毫秒为单位。
- `current_active_defrag_time`：自上次内存碎片超过限制以来经过的时间，以毫秒为单位。
- `tracking_total_keys`：服务器正在跟踪的 key 数量。
- `tracking_total_items`：正在跟踪的 item 数量，即每个 key 的客户端数量总和。
- `tracking_total_prefixes`：服务器前缀表(prefix table)中跟踪的前缀数（仅适用于广播模式）。
- `unexpected_error_replies`：意外错误回复的数量，即来自 AOF 加载或复制的错误类型。
- `total_error_replices`：发出的错误回复总数，即被拒绝的命令（命令执行之前的错误）和失败的命令（命令执行中的错误）的总和。
- `total_reads_processed`：处理的 read 事件总数。
- `total_writes_processed`：处理的 write 事件总数。
- `io_threaded_reads_processed`：主线程和 I/O 线程处理的 read 事件数量。
- `io_threaded_writes_processed`：主线程和 I/O 线程处理的 write 事件数量。

下面是所有 **replication** 相关信息：
- `role` ：如果实例不是任何节点的从节点，则值是 "master"，如果实例从某个节点同步数据，则值是 "slave"。请注意，一个从节点可以是另一个从节点的主节点（chained replication）。
- `master_failover_state`：正在进行的过账转移的状态（如果有）。
- `master_replid`：Redis 服务器的 replication ID。
- `master_replid2`：第二个 replication ID，用于故障转移后的 PSYNC。
- `master_repl_offset`：服务器的当前 replication 偏移量。
- `second_repl_offset`：接收 replication ID 的偏移量。
- `repl_backlog_active`：指示 replication backlog 处于活动状态的标志。
- `repl_backlog_size`：replication backlog 缓冲区的大小，以字节为单位。
- `repl_backlog_first_byte_offset`：replication backlog 缓冲区的主节点偏移量。
- `repl_backlog_histlen`：replication backlog 缓冲区中数据的大小，以字节为单位。

如果实例为从节点，则会提供一下额外字段：
- `master_host`：主节点的 Host 或 IP。
- `master_port`：主节点监听的 TPC 端口。
- `master_link_status`：连接状态 (up 或 down)。
- `master_last_io_seconds_ago`：自上次与主机诶单交互依赖，经过的秒数。
- `master_sync_in_progress`：指示主节点正在与从节点同步。（如果 SYNC 操作正在进行，则会提供以下这些字段）。
- `slave_repl_offset`：从节点实例(replica instance)的复制偏移量(replication offset)。
- `slave_priority`：实例作为故障转移候选者的优先级。
- `slave_read_only`：指示从节点是否为只读的标志。

如果 SYNC 操作正在进行，则会提供以下这些字段：
- `master_sync_total_bytes`：需要传输的字节总数。当大小未知时，这可能为 0（例如，当使用 `repl-diskless-sync` 配置指令时）。
- `master_sync_read_bytes`：已传输的字节数。
- `master_sync_left_bytes`：同步完成前剩余的字节数（当 `master_sync_total_bytes` 为 0 时可能为负）。
- `master_sync_perc`： `master_sync_read_bytes` 对于  `master_sync_total_bytes` 的百分比，或当 `master_sync_total_bytes` 为 0 时使用 `loading_rdb_used_mem` 的近似值。
- `master_sync_last_io_seconds_ago`：SYNC 操作期间自上次传输 I/O 以来的秒数。

如果 master 和 replica 之间的连接关闭，则会提供一个附加字段：
- `master_link_down_since_seconds`：自连接断开依赖，经过的秒数。

以下字段将始终提供：
- `connected_slaves`：已连接的从节点数。

如果服务器配置了 `min-slaves-to-write` (或从 Redis 5 开始使用 `min-replicas-to-write`) 指令，则提供一个附加字段：
- `min_slaves_good_slaves`：当前认为良好的副本数量。

对于每个从节点，将会添加以下行：
- `slaveXXX`：id, IP address, port, state, offset, lag

下面是所有 **cpu** 相关的信息：
- `used_cpu_sys`：Redis 服务器消耗的系统 CPU，即服务器进程所有线程（主线程和后台线程）消耗的系统 CPU 之和。
- `used_cpu_user`：Redis 服务器消耗的用户 CPU，即服务器进程所有线程（主线程和后台线程）消耗的用户 CPU 之和。
- `used_cpu_sys_children`：后台进程消耗的系统 CPU。
- `used_cpu_user_children`：后台进程消耗的用户 CPU。
- `used_cpu_sys_main_thread`：Redis 服务器主线程消耗的系统 CPU。
- `used_cpu_user_main_thread`：Redis 服务器主线程消耗的用户 CPU。

**commandstats** 部分提供基于命令类型的统计，包含调用（未拒绝）次数，这些命令消耗的总 CPU 时间、每次命令执行消耗的平均 CPU、被拒绝调用的次数（命令执行之前错误）、以及失败的调用次数（命令执行中的错误）。

对于每种命令类型，添加以下行：
- `cmdstat_XXX`：calls=XXX,usec=XXX,usec_per_call=XXX,rejected_calls=XXX,failed_calls=XXX

**errorstats** 部分可以根据回复错误前缀("-"之后的第一个单词，知道第一个空格。示例：ERR)来跟踪 Redis 中发送的不同错误。

对于每种错误类型，添加以下行：
- `errorstat_XXX`：count=XXX

**cluster** 部分目前仅包含一个唯一字段：
- `cluster_enabled`：表示 Redis Cluster 已启用。

如果 modules 提供的话，**modules** 部分包含有关加载 modules 的附加信息。本节中属性行的字段部分使用以 modules 的名称为前缀。

**keyspace** 部分统计有关每个数据库的主字典(main dictionary)的统计信息。统计信息是 key 的数量和过期的 key 数量。

对于每个数据库，添加以下行：
- `dbXXX`：keys=XXX,expires=XXX

**关于本手册页中使用的 slave 一词的说明：** 从 Redis 5 开始，如果不是为了向后兼容，Redis 项目不再使用 slave 一词。不幸的是，在这个命令中，slave 这个此是协议的一部分，所以只有当这个 API 自然被弃用时，我们才能删除此类事件。

**Modules generated sections：** 从 Redis 6 开始，模块可以将它们的信息注入 [INFO](info.md) 命令，即使提供了 all 参数，默认情况下也会排除这些部分（它将包含在已加载模块的列表，但不包括它们生成的信息字段）。要获得这些，您必须使用 `modules` 参数或 `everything`。