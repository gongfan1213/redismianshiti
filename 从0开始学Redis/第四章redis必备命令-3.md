
端信息，一行字符串对应一个已连接的客户端，而每行字符串由一系列“属性=值”形式的域组成，域与域之间用空格分开。

返回的域含义如下。

· id：表示客户端编号。

· addr：表示客户端的地址（IP 地址+端口）。

· fd：表示套接字所使用的文件描述符。

· age：表示已连接时长，单位为秒。

· idle：表示连接空闲时长，单位为秒。

· flags：表示客户端 flag（见下文）。

· db：指明该客户端正在使用的数据库，数值表示数据库的索引号。

· sub：表示已订阅的消息频道数量。

· psub：表示已订阅模式的数量。

· multi：表示在事务中被执行的命令数量。

· qbuf：表示查询缓冲区的长度，0 表示没有分配查询缓冲区，单位为字节。

· qbuf-free：表示查询缓冲区剩余空间的长度，0 表示没有剩余空间，单位为字节。

· obl：表示输出缓冲区的长度，0 表示没有分配输出缓冲区，单位为字节。

· oll：表示输出列表中包含的对象数量。如果输出缓冲区没有剩余空间，则命令回复会以字符串对象的形式被添加到这个队列中。

· omem：表示输出缓冲区和输出列表占用的内存总量。

· events：表示文件描述符事件。

> r：在 loop 事件中，表示套接字是可读的。

> w：在 loop 事件中，表示套接字是可写的。

· cmd：表示最近一次执行的命令。

客户端 flag 可以由以下几部分组成。

· O：客户端是 MONITOR 模式下的附属节点（slave）。

· S：客户端是一般模式下（normal）的附属节点。

· M：客户端是主节点（master）。

· x：客户端正在执行事务。

· b：客户端正在等待阻塞事件。

· i：客户端正在等待 VM I/O 操作（已废弃）。

· d：一个受监视（watched）的键已被修改，EXEC 命令将执行失败。

· c：在将回复完整地写出来之后，关闭连接。

· u：客户端未被阻塞（unblocked）。

· A：尽可能快地关闭连接。

· N：未设置任何 flag。

使用 CLIENT LIST 命令获取客户端相关信息，操作如下：
```js
127.0.0.1:6379> CLIENT LIST

id=3 addr=127.0.0.1:54547 fd=8 name= age=11736 idle=0 flags=N db=0 sub=0 psub=0

multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client #客户端3的信息
id=4 addr=127.0.0.1:55324 fd=9 name= age=5 idle=5 flags=N db=0 sub=0 psub=0 multi=-1
qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=client #客户端4的信息
```
## 2. CLIENT GETNAME 命令: 获取客户端名字

命令格式:

CLIENT GETNAME

CLIENT GETNAME 命令用于获取连接设置的名字。如果是新创建的连接，则它是没有名字的，CLIENT GETNAME 命令将会返回空值。

返回值: 如果连接没有设置名字，则命令执行后返回空值；如果连接设置了名字，则命令执行后将会返回这个名字。

使用 CLIENT GETNAME 命令获取客户端的名字，操作如下:

127.0.0.1:6379> CLIENT GETNAME
(nil) #表示客户端没有名字

## 3. CLIENT SETNAME 命令: 设置客户端名字

命令格式:

CLIENT SETNAME connection-name

CLIENT SETNAME 用于为当前连接设置一个名字。执行 CLIENT LIST 命令后的结果中会有这个连接的名字，它可以用来区分当前正在与服务器连接的客户端。使用字符串类型来保存这个连接的名字，它最多可以占用 512MB 的空间。注意，在为连接设置名字的时候，名字中要尽量避免出现空格。

可以通过将一个连接的名字设置为空字符串，来实现删除这个连接的名字。在默认情况下，新创建的连接没有名字。

返回值: CLIENT SETNAME 命令成功执行后，会返回 OK。

使用 CLIENT SETNAME 命令为客户端设置名字，操作如下:

```js
127.0.0.1:6379> CLIENT SETNAME MyRedis
OK
127.0.0.1:6379> CLIENT GETNAME
"MyRedis"
127.0.0.1:6379> CLIENT SETNAME MyRedis1
OK
127.0.0.1:6379> CLIENT GETNAME
"MyRedis1"
```
## 4. CLIENT PAUSE 命令: 在指定时间范围内停止运行来自客户端的命令

命令格式:

CLIENT PAUSH timeout

CLIENT PAUSH 命令用于在指定时间范围内停止运行来自客户端的命令。参数 timeout是一个正整数，以毫秒为单位。

返回值: CLIENT PAUSH 命令成功执行后返回 OK；如果 timeout 参数类型非法，则将会返回一个错误。

在指定时间范围内，使用 CLIENT PAUSE 命令停止运行来自客户端的命令，操作如下:
```js
127.0.0.1:6379> CLIENT PAUSE 100000
OK
127.0.0.1:6379> CLIENT GETNAME
"MyRedis1"
(96.12s)
```
## 5. CLIENT KILL 命令: 关闭客户端连接

命令格式:

CLIENT KILL ip:port

CLIENT KILL 命令用于杀死（关闭）一个客户端连接，ip:port 是这个客户端的 IP 地址和端口。Redis 是单线程的，当一个 Redis 命令正在执行时，不会有客户端被断开连接。

返回值: 如果 CLIENT KILL 命令中指定的客户端存在，并且该客户端连接被成功关闭，就返回 OK。

使用 CLIENT KILL 命令关闭客户端连接，操作如下:
```js
127.0.0.1:6379> CLIENT KILL 127.0.0.1:6379 #没有找到这个客户端，关闭失败
(error) ERR No such client
127.0.0.1:6379> CLIENT LIST #查看客户端信息
id=3 addr=127.0.0.1:54547 fd=8 name=MyRedis1 age=12561 idle=0 flags=N db=0 sub=0
psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client
id=4 addr=127.0.0.1:55324 fd=9 name= age=830 idle=830 flags=N db=0 sub=0 psub=0
multi=-1 qbuf=0 qbuf-free=0 obl=0 oll=0 omem=0 events=r cmd=command
127.0.0.1:6379> CLIENT KILL 127.0.0.1:55324 #关闭客户端连接
OK
127.0.0.1:6379> CLIENT KILL 127.0.0.1:54547
OK
127.0.0.1:6379> CLIENT LIST
id=5 addr=127.0.0.1:55351 fd=8 name= age=0 idle=0 flags=N db=0 sub=0 psub=0 multi=-1
qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client
```
## 4.5.2 查看 Redis 服务器信息
### 1. COMMAND 命令: 查看 Redis 命令的详细信息

命令格式:

COMMAND

COMMAND 命令以数组的形式返回 Redis 的所有命令的详细信息。

返回值: COMMAND 命令执行后，以随机列表的形式返回 Redis 的所有命令的详细信息。

使用 COMMAND 命令查看 Redis 命令的详细信息（由于篇幅有限，在这里只展示后面几个命令的信息），操作如下:
```js
177) 1) "discard"
2) (integer) 1
3) 1) "noscript"
2) "fast"
4) (integer) 0
5) (integer) 0
6) (integer) 0
178) 1) "hlen"
2) (integer) 2
3) 1) "readonly"
2) "fast"
4) (integer) 1
5) (integer) 1
6) (integer) 1
179) 1) "bitpos"
2) (integer) -3
3) 1) "readonly"
2) "fast"
4) (integer) 1
5) (integer) 1
6) (integer) 1
180) 1) "georadiusbymember_ro"
2) (integer) -5
3) 1) "readonly"
2) "movablekeys"
4) (integer) 1
5) (integer) 1
6) (integer) 1
```
## 2. COMMAND COUNT 命令: 统计 Redis 的命令个数

命令格式:

COMMAND COUNT

COMMAND COUNT 命令用于统计 Redis 的所有命令个数。


返回值: 返回一个数值，表示 Redis 的命令个数。

使用 COMMAND COUNT 命令统计 Redis 的命令个数，操作如下:

127.0.0.1:6379> COMMAND COUNT

(integer) 180

## 3. COMMAND GETKEYS 命令: 获取指定的所有键

命令格式:

COMMAND GETKEYS

COMMAND GETKEYS 命令用于获取指定的所有键。

返回值: COMMAND GETKEYS 命令执行后，返回键的列表。

使用 COMMAND GETKEYS 命令获取指定的键值对的所有键，操作如下:
```
127.0.0.1:6379> COMMAND GETKEYS MSET userName '刘河飞' age 22 sex '男' birthday
'1994-11-11' height 172
1) "userName"
2) "age"
3) "sex"
4) "birthday"
5) "height"
127.0.0.1:6379> COMMAND GETKEYS SET stuName '小明'
1) "stuName"
127.0.0.1:6379> COMMAND GETKEYS HSET student name '李回' age 23
1) "student"
```
## 4. COMMAND INFO 命令: 查看 Redis 命令的描述信息

命令格式:

COMMAND INFO key [key ...]

COMMAND INFO 命令用于获取 Redis 命令的描述信息。


返回值: COMMAND INFO 命令执行后，返回命令的描述信息，是一个嵌套的列表。

使用 COMMAND INFO 命令查看几个 Redis 命令的描述信息，操作如下:
```js
127.0.0.1:6379> COMMAND INFO
(empty list or set)
127.0.0.1:6379> COMMAND INFO SET SADD ZADD HSET LPUSH
1) 1) "set"
2) (integer) -3
3) 1) "write"
2) "denyoom"
4) (integer) 1
5) (integer) 1
6) (integer) 1
2) 1) "sadd"
2) (integer) -3
3) 1) "write"
2) "denyoom"
3) "fast"
4) (integer) 1
5) (integer) 1
6) (integer) 1
3) 1) "zadd"
2) (integer) -4
3) 1) "write"
2) "denyoom"
3) "fast"
4) (integer) 1
5) (integer) 1
6) (integer) 1
4) 1) "hset"
2) (integer) -4
3) 1) "write"
2) "denyoom"
3) "fast"
4) (integer) 1
5) (integer) 1
6) (integer) 1
5) 1) "lpush"
2) (integer) -3
3) 1) "write"
2) "denyoom"
3) "fast"
4) (integer) 1
5) (integer) 1
6) (integer) 1
```
## 5. DBSIZE 命令: 统计当前数据库中键的数量

命令格式:

DBSIZE

DBSIZE 命令用于统计当前数据库中键的数量。

返回值: 返回一个数值，表示当前数据库中键的数量。

使用 DBSIZE 命令统计 0 号、4 号数据库中键的数量，操作如下:
```js
127.0.0.1:6379> DBSIZE
(integer) 0 #0表示0号数据库为空，没有任何键
127.0.0.1:6379> MSET stuName '刘河飞' age 23 sex '男' height 172 weight 65
OK
127.0.0.1:6379> DBSIZE
(integer) 5
127.0.0.1:6379> SELECT 4 #切换到4号数据库
OK
127.0.0.1:6379[4]> DBSIZE
(integer) 8 #返回8，表示4号数据库中有8个键
```
## 6. INFO 命令: 查看服务器的各种信息

命令格式:

INFO [section]

INFO 命令用于查看 Redis 服务器的各种信息及统计相关数值。

参数 section 的设置可以让 INFO 命令只返回某一部分的信息。


INFO 命令执行后，会返回如下几部分的信息。

· server 部分：该部分主要说明 Redis 的服务器信息。

· clients 部分：该部分记录了已连接客户端的信息。

· memory 部分：该部分记录了 Redis 服务器的内存相关信息。

· persistence 部分：该部分记录了与持久化（RDB 持久化和 AOF 持久化）相关的信息。

· stats 部分：该部分记录了相关的统计信息。

· replication 部分：该部分记录了 Redis 数据库主从复制信息。

· cpu 部分：该部分记录了 CPU 的计算量统计信息。

· commandstats 部分：该部分记录了 Redis 各种命令的执行统计信息，如执行命令消耗的 CPU 时间、执行次数等。

· cluster 部分：该部分记录了与 Redis 集群相关的信息。

· keyspace 部分：该部分记录了与 Redis 数据库相关的统计信息，如键的数量。

参数 section 除可以取上面的值以外，它的值还可以是 all（表示返回所有信息）和 default（表示返回默认选择的信息 ）。

如果 INFO 命令不带任何参数，则默认以 default 作为参数。

以上各部分相关的详细信息，会在后面的章节中做补充说明。

返回值: 返回以上各部分相关的信息。

使用 INFO 命令查看服务器的详细信息，操作如下:
```
127.0.0.1:6379> INFO

# Server #服务器信息

redis_version:4.0.9
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:c706c7a026bd863d
redis_mode:standalone
os:Linux 2.6.32-431.el6.x86_64 x86_64
arch_bits:64
multiplexing_api:epoll
atomicvar_api:sync-builtin
gcc_version:4.4.7
process_id:18573
run_id:27719c508d875bd6462c6b768f691174f8431300
tcp_port:6379
uptime_in_seconds:381
uptime_in_days:0
hz:10
lru_clock:15484498
executable:/home/redis/redis-4.0.9/src/./redis-server
config_file:
# Clients #客户端信息
connected_clients:1
client_longest_output_list:0
client_biggest_input_buf:0
blocked_clients:0
# Memory #服务器内存信息
used_memory:853064
used_memory_human:833.07K
used_memory_rss:2682880
used_memory_rss_human:2.56M
used_memory_rss_ratio:3.14
used_memory_peak:853064
used_memory_peak_human:833.07K
used_memory_peak_perc:100.11%
used_memory_overhead:838038
used_memory_startup:786472
used_memory_dataset:15026
used_memory_dataset_perc:22.56%
total_system_memory:16725729280
total_system_memory_human:15.58G
used_memory_lua:37888
used_memory_lua_human:37.00K
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
mem_fragmentation_ratio:3.14
mem_allocator:jemalloc-4.0.3
active_defrag_running:0
lazyfree_pending_objects:0
# Persistence #服务器持久化信息
loading:0
rdb_changes_since_last_save:5
rdb_bgsave_in_progress:0
rdb_last_save_time:1542210773
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
rdb_last_cow_size:0
aof_enabled:0
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_write_status:ok
aof_last_cow_size:0
# Stats #服务器统计信息
total_connections_received:1
total_commands_processed:17
instantaneous_ops_per_sec:0
total_net_input_bytes:464
total_net_output_bytes:10369
instantaneous_input_kbps:0.00
instantaneous_output_kbps:0.00
rejected_connections:0
sync_full:0
sync_partial_ok:0
sync_partial_err:0
expired_keys:0
expired_stale_perc:0.00
expired_time_cap_reached_count:0
evicted_keys:0
keyspace_hits:0
keyspace_misses:0
pubsub_channels:0
pubsub_patterns:0
latest_fork_usec:0
migrate_cached_sockets:0
slave_expires_tracked_keys:0
active_defrag_hits:0
active_defrag_misses:0
active_defrag_key_hits:0
active_defrag_key_misses:0
# Replication #主从复制信息
role:master
connected_slaves:0
master_replid:d4dd011e969cc8b63514dfcc486aeda695039cbd
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:0
second_repl_offset:-1
repl_backlog_active:0
repl_backlog_size:1048576
repl_backlog_first_byte_offset:0
repl_backlog_histlen:0
# CPU #CPU统计信息
used_cpu_sys:0.20
used_cpu_user:0.10
used_cpu_sys_children:0.00
used_cpu_user_children:0.00
# Cluster #集群信息
cluster_enabled:0
# Keyspace #与数据库相关的统计信息
db0:keys=5,expires=0,avg_ttl=0
db4:keys=8,expires=0,avg_ttl=0
db5:keys=20,expires=1,avg_ttl=1498457788849227
db6:keys=2,expires=0,avg_ttl=0
```
## 7. LASTSAVE 命令: 获取最近一次保存数据的时间

命令格式:

LASTSAVE

LASTSAVE 命令用于获取最近一次 Redis 成功将数据保存到磁盘上的时间，时间格式是 UNIX 时间戳。

返回值: LASTSAVE 命令成功执行后，返回一个 UNIX 时间戳。

使用 LASTSAVE 命令查看服务器最近一次保存数据的时间，操作如下:
```js
127.0.0.1:6379> LASTSAVE
(integer) 1542210773
127.0.0.1:6379> SELECT 4
OK
127.0.0.1:6379[4]> LASTSAVE
(integer) 1542210773
```
## 8. MONITOR 命令: 实时打印服务器接收到的命令

命令格式:

MONITOR

MONITOR 命令常用于调试，它会实时打印出 Redis 服务器接收到的命令。

返回值: MONITOR 命令执行后总是返回 OK。

使用 MONITOR 命令实时打印服务器接收到的命令，操作如下:
```js
127.0.0.1:6379> MONITOR
OK
1542212130.398047 [4 127.0.0.1:55460] "SELECT" "0"
1542212138.829973 [0 127.0.0.1:55460] "KEYS" "*"
1542212151.198057 [0 127.0.0.1:55460] "COMMAND"
1542212160.319084 [0 127.0.0.1:55460] "GET" "name"
1542212167.477071 [0 127.0.0.1:55460] "TIME"
1542212180.917999 [0 127.0.0.1:55460] "TIME"
```
## 9. TIME 命令: 获取当前服务器的时间

命令格式:

TIME

TIME 命令用于获取当前服务器的时间。

返回值: TIME 命令执行后，会返回包含两个字符串的列表，第一个字符串是当前时间（UNIX 时间戳），第二个字符串是在当前这一秒内逝去的微秒数。

使用 TIME 命令查看当前服务器的时间，操作如下:
```js
127.0.0.1:6379> TIME
1) "1542212167"
2) "477084"
127.0.0.1:6379> TIME
1) "1542212180"
2) "918034"
```
## 4.5.3 修改并查看相关配置
### 1. CONFIG SET 命令: 修改 Redis 服务器的配置

命令格式:

CONFIG SET parameter value

CONFIG SET 命令用于修改 Redis 服务器的配置，修改之后不需要重新启动服务器也能生效。也可以使用该命令来修改 Redis 的持久化方式。 

能被 CONFIG SET 命令所修改的 Redis 配置参数都可以在配置文件 redis.conf 中找到。

返回值: 使用 CONFIG SET 命令修改 Redis 服务器配置成功时，返回 OK；否则返回一个错误。

使用 CONFIG SET 命令修改 Redis 服务器的配置信息，操作如下：
```js
127.0.0.1:6379> CONFIG SET slowlog-max-len 1000 #修改服务器慢日志的最大长度为1000个字符
OK
127.0.0.1:6379> CONFIG SET requirepass 123456 #修改服务器的登录密码为123456
OK
```
## 2. CONFIG GET 命令: 查看 Redis 服务器的配置

命令格式:

CONFIG GET parameter

CONFIG GET 命令用于获取运行中的 Redis 服务器的配置参数。

参数 parameter 是命令搜索关键字，用于查找所有匹配的配置参数，查找返回的参数和值以键值对的形式排列。

使用 CONFIG GET *命令来列出 Redis 服务器的所有配置参数及参数值。

使用 CONFIG GET a*命令来列出 Redis 服务器所有以 a 开头的配置参数及参数值。


返回值: CONFIG GER 命令执行后，返回给定配置参数的值。

使用 CONFIG GET 命令查看服务器的相关配置信息，操作如下：

127.0.0.1:6379> CONFIG GET requirepass
```js
1) "requirepass" #查看服务器的密码信息

2) "123456"
127.0.0.1:6379> CONFIG GET slowlog-max-len #查看服务器慢日志的最大长度
1) "slowlog-max-len"
2) "1000"
127.0.0.1:6379> CONFIG GET au* #查看AOF的相关配置
1) "auto-aof-rewrite-percentage"
2) "100"
3) "auto-aof-rewrite-min-size"
4) "67108864"
```

## 3. CONFIG RESETSTAT 命令: 重置 INFO 命令中的统计数据

命令格式:

CONFIG RESETSTAT

CONFIG Resetstat 命令用于重置 INFO 命令中的一些统计数据，具体如下。

· Keyspace hits：表示键空间命中次数。

· Keyspace misses：表示键空间不命中次数。

· Number of commands processed：表示执行命令的次数。

· Number of connections received：表示连接服务器的次数。

· Number of expired keys：表示过期 key 的数量。

· Number of rejected connections：表示被拒绝的连接数量。

· Latest fork(2) time：表示最后执行 fork(2)的时间。

· The aof_delayed_fsync counter：表示 aof_delayed_fsync 计数器的值。

返回值: CONFIG RESETSTAT 命令执行后，总是返回 OK。

使用 CONFIG RESETSTAT 命令重置 INFO 命令中的统计数据，操作如下：
```
127.0.0.1:6379> CONFIG RESETSTAT

OK
127.0.0.1:6379> INFO #再次使用INFO命令查看服务器信息
# Server
redis_version:4.0.9
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:c706c7a026bd863d
redis_mode:standalone
```
## 4. CONFIG REWRITE 命令: 改写 Redis 配置文件

命令格式:

CONFIG REWRITE

在 Redis 服务器启动时，会用到 redis.conf 文件，可以使用 CONFIG REWRITE 命令来修改这个文件。说简单一点，CONFIG REWRITE 命令的作用就是通过尽可能少的修改，尽量保留最初的配置，将服务器当前正在使用的配置信息保存到 redis.conf 文件中。

假设在启动 Redis 服务器时所指定的 redis.conf 文件不存在，则可以使用 CONFIG REWRITE 命令来重新构建并生成一个新的 redis.conf 文件。

在启动 Redis 服务器时，如果没有加载 redis.conf 文件，则在执行 CONFIG REWRITE 命令时将会报错。

注意: 对 redis.conf 文件的修改是原子性的，并且具有一致性，要么成功，要么失败。如果修改成功，那么 redis.conf 文件就是修改后的新文件。如果修改时出错，或者在修改时服务器崩溃，那么文件修改失败，最初的 redis.conf 文件不会被修改。

返回值: CONFIG REWRITE 命令如果修改配置文件成功，则返回 OK；修改失败，则返回一个错误。

使用 CONFIG REWRITE 命令改写 Redis 配置文件，操作如下：
```js
127.0.0.1:6379> CONFIG GET appendonly
1) "appendonly"
2) "yes"
127.0.0.1:6379> CONFIG SET appendonly no
OK
127.0.0.1:6379> CONFIG GET appendonly
1) "appendonly"
2) "no"
127.0.0.1:6379> CONFIG REWRITE #改写Redis配置文件
OK
```
# 4.5.4 数据持久化
## 1. BGREWRITEAOF 命令: 执行 AOF 文件重写操作

命令格式:

BGREWRITEAOF

BGREWRITEAOF 命令用于执行一个 AOF 文件重写操作。该命令执行后，会创建一个当前 AOF 文件的优化版本。当 BGREWRITEAOF 命令执行失败时，AOF 文件数据并不会丢失。旧的 AOF 文件在 BGREWRITEAOF 命令执行成功之前是不会被修改的，因此不存在数据丢失问题。

在 Redis 的高版本（Redis 2.4 以后）中，AOF 文件的重写将由 Redis 自动触发，使用 BGREWRITEAOF 命令只是用于手动触发重写操作。

返回值: BGREWRITEAOF 命令执行后，返回反馈的信息。

使用 BGREWRITEAOF 命令执行 AOF 文件重写操作，操作如下：
```js
127.0.0.1:6379> BGREWRITEAOF

18955:M 15 Nov 01:06:47.927 * Background append only file rewriting started by pid 18961
Background append only file rewriting started
```
## 2. SAVE 命令: 将数据同步保存到磁盘中

命令格式:

SAVE

SAVE 命令用于保存数据到磁盘中。SAVE 命令具体执行的是一个同步保存操作，它以 RDB 文件的形式将当前 Redis 的所有数据快照保存到磁盘中。在生产环境中，不建议使用 SAVE 命令来保存数据，因为它在执行后会阻塞所有客户端。推荐使用 BGSAVE 命令来异步执行保存数据的任务。如果后台子进程保存数据失败，或者出现其他问题，则可以使用 SAVE 命令来做最后的保存。

返回值: 当 SAVE 命令成功执行时，也就是保存数据成功时，会返回 OK。

执行 SAVE 命令将数据库中的数据保存到磁盘中，操作如下：
```js
127.0.0.1:6379> SAVE

18955:M 15 Nov 01:08:48.059 * DB saved on disk
OK
```
## 3. BGSAVE 命令: 将数据异步保存到磁盘中

命令格式:

BGSAVE

BGSAVE 命令用于在 Redis 服务后端采用异步的方式将数据保存到当前数据库的磁盘中。

BGSAVE 命令的执行原理: 在 BGSAVE 命令执行后会返回 OK，之后 Redis 启动一个新的子进程，原来的 Redis 进程（父进程）继续执行客户端请求操作，而子进程则负责将数据保存到磁盘中，然后退出。


我们可以使用 LASTSAVE 命令来查看相关信息，进而判断 BGSAVE 命令是否将数据保存成功。

使用 BGSAVE 命令将数据异步保存到磁盘中，操作如下：
```js
127.0.0.1:6379> BGSAVE
18955:M 15 Nov 01:10:04.661 * Background saving started by pid 18978
Background saving started
```
# 4.5.5 实现主从服务
## 1. SYNC 命令

命令格式:

SYNC

SYNC 命令是 Redis 复制功能的内部命令，在后面的章节中将会讲述。

返回值: SYNC 命令没有明确的返回值。

执行 SYNC 命令，操作如下：
```js
127.0.0.1:6379> SYNC
Entering slave output mode... (press Ctrl-C to quit)
18955:M 15 Nov 01:14:01.560 * Slave 127.0.0.1:<unknown-slave-port> asks for synchronization
18955:M 15 Nov 01:14:01.561 * Starting BGSAVE for SYNC with target: disk
18955:M 15 Nov 01:14:01.561 * Background saving started by pid 19320
19320:C 15 Nov 01:14:01.563 * DB saved on disk
19320:C 15 Nov 01:14:01.564 * RDB: 0 MB of memory used by copy-on-write
18955:M 15 Nov 01:14:01.631 * Background saving terminated with success
SYNC with master, discarding 1595 bytes of bulk transfer...
18955:M 15 Nov 01:14:01.632 * Synchronization with slave 127.0.0.1:<unknown-slave-
port> succeeded
SYNC done. Logging commands from master.
"PING"
"PING"
"PING"
```
## 2. PSYNC 命令

命令格式:

PSYNC <master_run_id> <offset>

PSYNC 命令也是 Redis 复制功能的内部命令。

返回值: PSYNC 命令没有明确的返回值。

执行 PSYNC 命令，操作如下：
```js
[root@localhost src]# ./redis-cli

127.0.0.1:6379> PSYNC
Entering slave output mode... (press Ctrl-C to quit)
18955:M 15 Nov 01:15:47.202 * Slave 127.0.0.1:<unknown-slave-port> asks for synchronization
18955:M 15 Nov 01:15:47.202 * Starting BGSAVE for SYNC with target: disk
19328:C 15 Nov 01:15:47.203 * Background saving started by pid 19328
19328:C 15 Nov 01:15:47.205 * DB saved on disk
18955:M 15 Nov 01:15:47.253 * RDB: 0 MB of memory used by copy-on-write
18955:M 15 Nov 01:15:47.253 * Background saving terminated with success
SYNC with master, discarding 1596 bytes of bulk transfer...
ERROR,"ERR wrong number of arguments for 'psync' command"
"PING"
"PING"
"PING"
```
## 3. SLAVEOF 命令: 修改复制功能

命令格式:

SLAVEOF host port

在 Redis 运行时，可以使用 SLAVEOF 命令动态修改复制功能的行为。我们利用 SLAVEOF host port 命令来修改当前服务器，使其转变为指定服务器的从属服务器”(Slave Server）。如果当前服务器是某个主服务器的从属服务器，则在执行 SLAVEOF host port 命令后，会使当前服务器停止对旧主服务器的同步，并且将旧数据集丢弃，然后开始对新主服务器数据进行同步。

如果想在同步时不丢失数据集，则可以使用 SLAVEOF NO ONE 命令，该命令执行后不会丢弃同步数据集。当主服务器出现故障的时候，我们可以利用该命令将从属服务器用作新的主服务器，实现数据不丢失，不间断运行。

返回值: SLAVEOF 命令执行后，总是返回 OK。

使用 SLAVEOF 命令修改服务器的复制行为，操作如下：
```js
127.0.0.1:6379> SLAVEOF 127.0.0.1 6380

OK
127.0.0.1:6379>SLAVEOF NO ONE
OK
```
## 4. ROLE 命令: 查看主从服务器的角色

命令格式:

ROLE

ROLE 命令用于查看主从服务器的角色，角色有 master、slave、sentinel。

返回值: ROLE 命令成功执行后返回一个数组，该数组中记录的是主从服务器的角色。

使用 ROLE 命令查看主从服务器的角色，操作如下：
```js
127.0.0.1:6379> ROLE
1) "master"
2) (integer) 0
3) (empty list or set)
```
## 4.5.6 服务器管理
### 1. SLOWLOG 命令: 管理 Redis 的慢日志

命令格式:

SLOWLOG subcommand [argument]

Slow log（慢日志）是 Redis 的日志系统，用于记录查询执行时间。查询执行时间指的是执行一个查询命令所耗费的时间，它不包括客户端响应、发送信息等 I/O 操作。Slow log 保存在内存里面，读/写速度非常快。

返回值: SLOWLOG 命令有多个子命令，每个子命令的作用不一样，返回值也不同。关于 SLOWLOG 命令的其他相关用法，在后面的章节中会详细讲解。

使用 SLOWLOG 命令管理 Redis 的慢日志，操作如下：
```js
127.0.0.1:6379> SLOWLOG GET 3 #查看日志信息

1) 1) (integer) 0
2) (integer) 1542211154
3) (integer) 11561
4) "INFO"
5) "127.0.0.2:55460"
6) ""
127.0.0.1:6379> SLOWLOG SET 2 #错误命令
(error) ERR Unknown SLOWLOG subcommand or wrong # of args. Try GET, RESET, LEN
127.0.0.1:6379> SLOWLOG LEN #查看当前慢日志的数量
(integer) 1
127.0.0.1:6379> SLOWLOG RESET #清空慢日志
OK
127.0.0.1:6379> SLOWLOG GET 3
(empty list or set)
```
## 2. SHUTDOWN 命令: 关闭 Redis 服务器或客户端

命令格式:

SHUTDOWN [SAVE|NOSAVE]

SHUTDOWN 命令具有多种作用，具体如下：

· 直接关闭 Redis 服务器。

· 关闭（停止）所有客户端。

· 在 AOF 选项被打开的情况下，执行 SHUTDOWN 命令将会更新 AOF 文件。

· 如果 Redis 服务中至少存在一个保存点在等待，则在执行 SHUTDOWN 命令的同时将会执行 SAVE 命令。

在持久化被打开的情况下，执行 SHUTDOWN 命令，它会保证服务器正常关闭，不会丢失任何数据。

SHUTDOWN SAVE 命令会强制 Redis 数据库执行保存命令，即使没有设置保存点，也会执行。

SHUTDOWN NOSAVE 命令的作用与 SHUTDOWN SAVE 命令的作用刚好相反，它会阻止 Redis 数据库执行保存操作，即使设置了一个或多个保存点，也会阻止。

返回值: SHUTDOWN 命令执行失败时返回错误；执行成功时什么信息也不返回，此时服务器和客户端的连接会断开，同时客户端自动退出。

使用 SHUTDOWN 命令关闭 Redis 服务器或客户端，操作如下：

```js
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> SHUTDOWN #关闭Redis服务器或客户端
18573:M 15 Nov 01:02:48.572 # User requested shutdown...
18573:M 15 Nov 01:02:48.573 * Saving the final RDB snapshot before exiting.
18573:M 15 Nov 01:02:48.574 * DB saved on disk
18573:M 15 Nov 01:02:48.574 * Redis is now ready to exit, bye bye...
not connected> PING
Could not connect to Redis at 127.0.0.1:6379: Connection refused
not connected> exit
[1]+ Done ./redis-server
[root@localhost src]# ps -ef|grep redis #查看Redis的进程，说明Redis服务器、客户端已被关闭
root 18943 18550 0 01:03 pts/0 00:00:00 grep redis
```
到这里为止，我们为大家介绍完了与 Redis 数据库相关的必备命令，希望大家多动手实践，熟练掌握相关命令，为以后的工作提供帮助。 
