
127.0.0.1:6379[5]> TYPE student

#键student是哈希类型的

hash

# 4.1.6 删除键
## 1. DEL 命令: 删除键

命令格式:

DEL key [key ...]

DEL 命令用于删除给定的一个或多个 key。如果这个 key 不存在，则会被忽略。

返回值: DEL 命令成功执行后，返回被删除 key 的数量。

删除 5 号数据库中的学生信息，操作如下:

127.0.0.1:6379[5]> KEYS *

1) "age-set"

2) "student"

3) "read"

4) "red"

5) "article1"

6) "age"

7) "massage"

8) "name"

9) "really"

10) "mobile"

11) "reduce"

12) "className"

13) "class-name-1"

14) "girl-name"

15) "userName"

16) "redis"

17) "score-list"

127.0.0.1:6379[5]> DEL student

(integer) 1

127.0.0.1:6379[5]> DEL age name userName

(integer) 3

## 2. PERSIST 命令: 删除键的生存时间

命令格式:


PERSIST key

PERSIST 命令用于删除给定 key 的生存时间，将这个带有生存时间的 key 转化为一个不带生存时间的永久 key（永不过期）。

返回值: 当 PERSIST 命令成功删除给定 key 的生存时间时，返回 1；如果 key 不存在，或者 key 并没有生存时间，则返回 0，表示该命令执行失败。

先设置学生的姓名、年龄的生存时间，然后删除它们的生存时间，操作如下:

```js
127.0.0.1:6379[5]> SET name '赵小雨'
OK
127.0.0.1:6379[5]> SET age 23
OK
127.0.0.1:6379[5]> EXPIRE name 180 #设置name键的过期时间为180秒
(integer) 1
127.0.0.1:6379[5]> EXPIRE age 160 #设置age键的过期时间为160秒
(integer) 1
127.0.0.1:6379[5]> TTL name
(integer) 161
127.0.0.1:6379[5]> TTL age
(integer) 149
127.0.0.1:6379[5]> PERSIST name#删除name键的过期时间
(integer) 1
127.0.0.1:6379[5]> TTL name #获取name键的过期时间，为-1表示name键的过期时间已被删除
(integer) -1
127.0.0.1:6379[5]> PERSIST age #删除age键的过期时间
(integer) 1
127.0.0.1:6379[5]> TTL age #获取age键的过期时间，为-1表示age键的过期时间已被删除
(integer) -1
```
## 4.2 HyperLogLog 命令

HyperLogLog 是 Redis 用来做基数统计的算法。当 Redis 数据库中的数据量非常庞大时，使用 HyperLogLog 命令来计算相关基数时，它具有所需空间固定、所占空间小的优点。在 Redis 中，每个 HyperLogLog 键只需要耗费 12KB 的内存，就可以计算接近 2^64 个不同元素的基数。HyperLogLog 不会存储输入的元素，它仅仅根据输入的元素来计算基数，因此它不会返回输入的元素。

这时，聪明的读者也许会问: 到底什么是基数？基数有什么特点呢？

举一个例子，有数据集{1,3,5,8,5,8,9}，去掉重复数据之后，得到这个数据集的基数集为{1,3,5,8,9}，这个基数集的基数就是 5。

基数的特点: 基数不可重复，且基数估计在误差允许的范围内。

我们使用学生信息表 4.1 作为实例，来介绍 HyperLogLog 命令，具体如下。

### 4.2.1 添加键值对到 HyperLogLog 中


PFADD 命令: 向 HyperLogLog 中添加键值对。

命令格式:

PFADD key element [element ...]

PFADD 命令用于将一个或多个指定的 key 添加到 HyperLogLog 中。HyperLogLog 内部可能会更新添加进来的 key，来反映一个不同的唯一元素估计数量，这个数量就是集合的基数。

返回 0。如果命令在执行时，这个给定的 key 不存在，那么命令会返回 1；否则返回HyperLogLog，再执行命令。

在执行 PFADD 命令时，我们可以只设置 key，而不设置这个 key 所对应的元素。PFADD 命令在执行时，如果给定的键（key）已经是一个 HyperLogLog，那么这个命令将什么也不做；如果给定的键（key）不存在，那么该命令会先创建一个空的 HyperLogLog，再返回 1。

返回值: PFADD 命令成功执行后，如果 HyperLogLog 的内部更新了，那么返回 1；否则返回 0。

使用 PFADD 命令将学生的姓名、年龄分别添加到 name-log、age-log 中，操作如下:

```js
127.0.0.1:6379[5]> PFADD name-log '刘河飞' '赵小雨' '武恬' '周明' '王丽'
(integer) 1
127.0.0.1:6379[5]> PFADD age-log 23 24 25 22 23
(integer) 1
```
## 4.2.2 获取 HyperLogLog 的基数

PFCOUNT 命令: 获取 HyperLogLog 的基数。

命令格式:

PFCOUNT key [key ...]

PFCOUNT 命令用于返回 HyperLogLog 的近似基数。当给定的 key 只有一个时，返回存储在给定键的 HyperLogLog 的近似基数；如果 key 不存在，则返回 0。当给定的 key 有多个时，PFCOUNT 命令返回给定 HyperLogLog 的并集的近似基数，这个近似基数是通过将所有给定的 HyperLogLog 合并到一个临时的 HyperLogLog 中计算出来的。

利用 HyperLogLog，用户可以使用较小的固定大小的内存来存储集合中的唯一元素。每个 HyperLogLog 只需要使用 12KB 的内存，以及几个字节的内存来存储键本身。

使用 PFCOUNT 命令获得的基数并不是准确的，它是一个带有 0.81%标准错误的近似值。

返回值: PFCOUNT 命令成功执行后，返回一个整数，这个整数是给定 HyperLogLog 包含的唯一元素的近似数量。

使用 PFCOUNT 命令获取添加到 HyperLogLog 中的键的基数，操作如下:

```js
127.0.0.1:6379[5]> PFCOUNT name-log
(integer) 5
127.0.0.1:6379[5]> PFCOUNT age-log
(integer) 4
127.0.0.1:6379[5]> PFCOUNT score-log
(integer) 0
```
## 4.2.3 合并 HyperLogLog
PFMERGE 命令: 合并多个 HyperLogLog 为一个新的 HyperLogLog。

命令格式:

PFMERGE destkey sourcekey [sourcekey ...]

PFMERGE 命令用于将多个 HyperLogLog 合并为一个新的 HyperLogLog，这个新的HyperLogLog 的基数接近于所有输入 HyperLogLog 的可见集合的并集。

合并之后的 HyperLogLog 将被存储在 destkey 键中。在执行 PFMERGE 命令之前，先检查 destkey 键是否存在，如果不存在，则会先创建一个空的 HyperLogLog，再执行命令。

返回值: PFMERGE 命令成功执行后，将会返回 OK。

将学生年龄、分数分别添加到 HyperLogLog 中，然后合并它们，操作如下:

```js
127.0.0.1:6379[5]> PFADD age-logs 23 24 25 22 23 #添加键值对到HyperLogLog中
(integer) 1
127.0.0.1:6379[5]> PFADD score-log 98 87 94 63 77
(integer) 1
127.0.0.1:6379[5]> PFMERGE age-score age-logs score-log #合并age-logs、score-log到age-score中
OK
127.0.0.1:6379[5]> PFCOUNT age-score #获取age-score的基数
(integer) 9
127.0.0.1:6379[5]> PFCOUNT age-logs score-log age-score
(integer) 9
```
## 4.3 脚本命令

Redis 脚本使用 Lua 解释器来执行。使用 Redis 脚本可以一次性将多个请求命令发送出去，以减少网络的开销；使用 Redis 脚本实现原子操作，Redis 会将整个脚本作为一个整体执行，中间不会有其他命令被执行，以此来保证原子性；使用 Redis 脚本可以达到复用的目的，因为 Redis 会永久保存客户端发送的脚本，所以其他客户端可以直接复用这个脚本。

Redis 脚本命令用于操作 Redis 脚本。

下面为大家介绍与 Redis Lua 脚本相关的几个命令。

### 4.3.1 缓存中的 Lua 脚本

1. SCRIPT LOAD 命令: 添加 Lua 脚本到缓存中

命令格式:

SCRIPT LOAD script

SCRIPT LOAD 命令用于将脚本 script 添加到脚本缓存中，但是并不会立即执行这个脚本。该命令与 EVAL 命令相似，但是 EVAL 命令在将脚本添加到缓存中后，会立即对输入的脚本进行求值操作。如果给定的脚本已经在缓存中存在，那么这个命令什么也不做。脚本被加入缓存中以后，可以通过 EVALSHA 命令使用脚本的 SHA1 校验和来调用这个脚本。

Lua 脚本可以在 Redis 的缓存中长时间保存，直到遇到 SCRIPT FLUSH 命令为止。

返回值: SCRIPT LOAD 命令成功执行后，返回给定 script 的 SHA1 校验和。

切换到 6 号数据库，将多个脚本添加到数据库中，操作如下:
```js
127.0.0.1:6379[6]> SCRIPT LOAD "return {20-6, 3+4*8, 'hello redis'}"
"d224a65120a5e3360132b84dbb42a756c2ff6f6d8"
127.0.0.1:6379[6]> SCRIPT LOAD "return {1+2+3+4, 8-(3+4)/2+7, (2*4)/3, 'good luck'}"
"98b71fff89d44a50a32772c7601fa8b43ba0a418c"
127.0.0.1:6379[6]> SCRIPT LOAD "return {abc + bcd, a * 3, bcdef + efd}"
"1dc86b853ee4dd9dd3f41f27c218fa074fdbc5888"
```
## 2. SCRIPT EXISTS 命令: 判断脚本是否已在缓存中

命令格式:

SCRIPT EXISTS sha1 [sha1 ...]

SCRIPT EXISTS 命令用于判断给定的一个或多个脚本的 SHA1 校验和指定的脚本是否已经被保存到 Redis 的缓存中。

返回值: SCRIPT EXISTS 命令成功执行后，返回一个列表，包含 0 和 1，0 表示缓存中不存在脚本，1 表示缓存中存在脚本。返回的这个列表中的元素和给定的 SHA1 校验和一一对应。

判断上面添加的 3 个 Lua 脚本是否已在缓存中，操作如下:

```js
127.0.0.1:6379[6]> SCRIPT EXISTS 'd224a65120a5e3360132b84dbb42a756c2ff6f6d8'
1) (integer) 1
127.0.0.1:6379[6]> SCRIPT EXISTS '98b71fff89d44a50a32772c7601fa8b43ba0a418c'
1) (integer) 1
127.0.0.1:6379[6]> SCRIPT EXISTS '1dc86b853ee4dd9dd3f41f27c218fa074fdbc5888'
1) (integer) 1
127.0.0.1:6379[6]> SCRIPT EXISTS '1dc86b853ee4dd9dd3f41f27c218fa074fdbc5999'
1) (integer) 0
```
## 4.3.2 对 Lua 脚本求值
### 1. EVAL 命令: 对 Lua 脚本求值

命令格式:

EVAL script numkeys key [key ...] arg [arg ...]

EVAL 命令用于对 Lua 脚本求值。在 Redis 的高版本（2.6.0 以后）中，嵌入了 Lua 解释器，使得 Redis 可以操作 Lua 脚本。

· 参数 script 是一段 Lua 脚本程序，它运行在 Redis 的服务器中。

· 参数 numkeys 用于指定键名参数的个数。

· 键名参数 key [key...]从 EVAL 命令的第三个参数开始算起，表示脚本中所用到的那些 Redis 键（key）。在 Lua 脚本中，可以使用全局变量 KEYS 数组（下标从 1 开始）来访问这些键名参数。

· 参数 arg [arg ...]是附加参数。在 Lua 脚本中，可以使用 ARGV 数组（下标从 1 开始）来访问这些附加参数。

与 Lua 脚本语言相关的知识，请读者自行查阅学习。

使用 EVAL 命令对 Lua 脚本求值，操作如下:

```js
127.0.0.1:6379[6]> SET score-sum '80+90,100+90'
OK
127.0.0.1:6379[6]> EVAL "return redis.call('get', 'score-sum')" 1
(error) ERR Number of keys can't be greater than number of args
127.0.0.1:6379[6]> EVAL "return redis.call('get', 'score-sum')" 0
"80+90,100+90"
127.0.0.1:6379[6]> EVAL "return {2*7+2-3, 100/3.0, 34/2-7, {9, 'hello redis'}}" 0
1) (integer) 13
2) (integer) 33
3) (integer) 10
4) (integer) 9
2) "hello redis"
127.0.0.1:6379[6]> EVAL "return {KEYS[1], KEYS[2], ARGV[1], ARGV[2]}" 2 name age '刘河飞' 23
1) "name"
2) "age"
3) "\xe5\x88\x98\xe6\xb2\xab\x3\xe9\xa3\x9e"
4) "23"
127.0.0.1:6379[6]> EVAL "return redis.call('PING')" 0 #Lua脚本执行PING命令
PONG
```
## 2. EVALSHA 命令: 对缓存中的脚本求值




命令格式:

EVALSHA sha1 numkeys key [key ...] arg [arg ...]

EVALSHA 命令用于根据给定的 SHA1 校验码来对缓存在服务器中的脚本求值。

使用 SCRIPT LOAD 命令将脚本缓存到服务器中。

使用 EVALSHA 命令对前面添加的 3 个 Lua 脚本求值，操作如下:
```js
127.0.0.1:6379[6]> EVALSHA d224a65120a5e3360132b84dbb42a756c2ff6f6d8 0
1) (integer) 14
2) (integer) 20
3) "hello redis"
127.0.0.1:6379[6]> EVALSHA 98b71fff89d44a50a32772c7601fa8b43ba0a418c 0
1) (integer) 10
2) (integer) 9
3) (integer) 2
3) "good luck"
127.0.0.1:6379[6]> EVALSHA 1dc86b853ee4dd9dd3f41f27c218fa074fdbc5888 0 #Lua 脚本存在语法错误
(error) ERR Error running script (call to f_1dc86b853ee4dd9dd3f41f27c218fa074fdbc5888):
@enable_strict_lua:15: user_script:1: Script attempted to access nonexistent global
variable 'abc'
127.0.0.1:6379[6]> EVALSHA 1dc86b853ee4dd9dd3f41f27c218fa074fdbc5999 0 #对不存在的Lua脚本求值
(error) NOSCRIPT No matching script. Please use EVAL.
```
## 4.3.3 杀死或清除 Lua 脚本
### 1. SCRIPT KILL 命令: 杀死正在运行的 Lua 脚本

命令格式:

SCRIPT KILL

针对一个正在运行且没有执行过任何写操作的 Lua 脚本，可以使用 SCRIPT KILL 命令来杀死它。SCRIPT KILL 命令主要用于终止运行时间过长的脚本。该命令执行之后，当前正在运行的脚本会被杀死，执行这个脚本的客户端会从 EVAL 命令的阻塞当中退出，并收到一个错误的返回值。

如果这个正在运行的脚本执行过写操作，那么使用 SCRIPT KILL 命令是无法杀死它的。Lua 脚本是原子性执行的。如果你非要杀死这个运行中的 Lua 脚本，则可以使用 SHUTDOWN NOSAVE 命令来直接关闭整个 Redis 进程，进而停止这个脚本的运行，并防止不完整的数据写入数据库中。

返回值: 如果 SCRIPT KILL 命令成功杀死这个脚本，就返回 OK；相反，如果失败，则返回一个错误。

杀死前面添加并正在运行的 Lua 脚本，操作如下:
```js
127.0.0.1:6379[6]> SCRIPT KILL
(error) NOTBUSY No scripts in execution right now. #表示没有脚本在运行
127.0.0.1:6379[6]> SCRIPT KILL
OK #表示成功杀死脚本
127.0.0.1:6379[6]> SCRIPT KILL
(error) ERR Sorry the script already executed write commands against the dataset.
You can either wait the script termination or kill the server in an hard way using the
SHUTDOWN NOSAVE command. #表示想杀死一个已经完成写操作的Lua脚本，失败
```
### 2. SCRIPT FLUSH 命令: 清除缓存中的 Lua 脚本

命令格式:

SCRIPT FLUSH

SCRIPT FLUSH 命令用于清除 Redis 服务器中的所有 Lua 脚本缓存。

返回值: 总是返回 OK。

使用 SCRIPT FLUSH 命令清除缓存中的 Lua 脚本，操作如下:
```js
127.0.0.1:6379[6]> SCRIPT EXISTS d224a65120a5e3360132b84dbb42a756c2ff6f6d8
1) (integer) 1 #表示缓存中存在该Lua脚本
127.0.0.1:6379[6]> SCRIPT EXISTS '98b71fff89d44a50a32772c7601fa8b43ba0a418c'
1) (integer) 1
127.0.0.1:6379[6]> SCRIPT FLUSH #清除Lua脚本
OK
127.0.0.1:6379[6]> SCRIPT EXISTS d224a65120a5e3360132b84dbb42a756c2ff6f6d8
1) (integer) 0 #Lua脚本清除成功
127.0.0.1:6379[6]> SCRIPT EXISTS '98b71fff89d44a50a32772c7601fa8b43ba0a418c'
1) (integer) 0
```
# 4.4 连接命令

Redis 连接命令主要用于连接 Redis 的服务，如查看服务状态、切换数据库等，具体如下。

### 4.4.1 解锁密码


AUTH 命令: 用于解锁密码。

命令格式:

AUTH password

我们通过修改 Redis 配置文件中 requirepass 项的值，来为 Redis 设置密码，进而使用密码来保护 Redis 服务器。命令为:

CONFIG SET requirepass password

在成功设置密码之后，我们每次连接 Redis 服务器都需要使用 AUTH 命令来解锁密码，解锁成功之后才能使用 Redis 的其他命令。如果 AUTH 命令使用的密码 password 和配置文件设置的密码相同，则服务器会返回 OK，同时开始接收其他命令的输入。如果输入的密码错误或者不匹配，则服务器会返回一个错误，并要求客户端重新输入。

返回值: 当 AUTH 命令给定的密码与配置文件设置的密码匹配成功时，服务器将会返回 OK；匹配失败会返回一个错误。

为数据库设置密码，并使用 AUTH 命令解锁密码，操作如下:
```js
127.0.0.1:6379> CONFIG SET requirepass 123456 #设置数据库的密码为123456
OK
127.0.0.1:6379> EXIT
[root@localhost src]#./redis-cli
127.0.0.1:6379>
(error) NOAUTH Authentication required.
127.0.0.1:6379> AUTH 123456 #使用AUTH命令解锁密码
OK
127.0.0.1:6379> SET key value
OK
127.0.0.1:6379> CONFIG GET requirepass #查看数据库的密码
1) "requirepass"
2) "123456"
```
可以使用 CONFIG SET requirepass "" 命令来将密码设置为空，也就是清空密码。
### 4.4.2 断开客户端与服务器的连接

QUIT 命令: 断开客户端与服务器的连接。

命令格式:

QUIT

QUIT 命令用于断开当前客户端与服务器的连接。一旦所有等待中的回复顺序写入客户端，这个连接就会被断开。

返回值: QUIT 命令执行后，总是返回 OK。

使用 QUIT 命令断开客户端与服务器的连接，操作如下:
```js
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> QUIT
[root@localhost src]#
```
## 4.4.3 查看服务器的运行状态

PING 命令: 查看服务器的运行状态。

命令格式:

PING

PING 命令主要用于查看 Redis 服务器是否正常运行。我们使用客户端向 Redis 服务器发送一个 PING 命令，如果服务器正常运行，则会返回一个 PONG。我们常常使用 PING 命令来测试客户端与服务器的连接是否正常，或者用于测量服务器的延迟值。

返回值: 如果客户端与服务器连接正常，就返回一个 PONG；否则返回一个错误。

使用 PING 命令查看服务器的运行状态，操作如下:
```js
127.0.0.1:6379> PING

PONG #说明服务器正常运行

127.0.0.1:6379> PING
Could not connect to Redis at 127.0.0.1:6379: Connection refused #说明服务器已被杀
死(kill)
not connected>
```
### 4.4.4 输出打印消息

ECHO 命令: 输出打印消息。

命令格式:

ECHO message

ECHO 命令用于输出打印消息，主要在测试时使用。

返回值: ECHO 命令成功执行后，将会返回 message（消息）。

使用 ECHO 命令输出打印消息，操作如下:
```js
127.0.0.1:6379> ECHO 'hello world'
"hello world"
127.0.0.1:6379> ECHO 1+2
"1+2"
127.0.0.1:6379> ECHO 3.141592657431652
"3.141592657431652"
```
## 4.4.5 切换数据库

SELECT 命令: 切换数据库。

命令格式:

SELECT index

SELECT 命令用于切换数据库，其中 index 是数字值，是数据库的索引，从 0 开始。默认使用 0 号数据库。

返回值: SELECT 命令执行后总是返回 OK，表示数据库切换成功。

使用 SELECT 命令切换数据库，操作如下:
```js
127.0.0.1:6379> SELECT 0

OK
127.0.0.1:6379> SELECT 2
OK
127.0.0.1:6379[2]> SELECT 16 #Redis数据库默认从0号到15号，超过了将会报错
(error) ERR DB index is out of range
127.0.0.1:6379[2]> SELECT 15
OK
```
## 4.5 服务器命令

Redis 服务器命令主要用于操作管理 Redis 服务，比如，管理 Redis 的日志，保存数据，修改相关配置等。

Redis 服务器命令具体如下。

### 4.5.1 管理客户端

1. CLIENT LIST 命令: 获取客户端相关信息

命令格式:

CLIENT LIST

CLIENT LIST 命令用于获取所有连接到服务器的客户端信息和统计数据。
返回值: CLIENT LIST 命令执行后，会以字符串形式打印输出所有连接到服务器的客
