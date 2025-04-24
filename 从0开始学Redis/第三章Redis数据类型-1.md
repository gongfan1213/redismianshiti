### 第3章 Redis数据类型

前面为大家介绍了Redis的特性、使用场景、安装及相关客户端，本章着重讲解与Redis的数据类型相关的命令。目前Redis数据库支持5种数据类型，分别是String（字符串）、Hash（哈希）、List（列表）、Set（集合）及Sorted Set（有序集合）。接下来我们逐个讲解其相关命令操作。

需要说明的是，Redis命令名称的大小写并不会影响命令表的查找结果。Redis的命令表使用的是与大小写无关的查找算法，输入的命令名称只要是正确的，无论大小写，都能得到正确的结果，命令表就能返回相同的redisCommand结构。关于命令表和redisCommand结构的详细说明参见后面的章节。在后面的相关命令的操作过程中，不严格区分大小写。

注意：在获取键值对中的值时，如果值是中文的，则会返回编码后的字符串。如果你希望返回值是中文的，那么在客户端连接服务器端时，可以使用命令“redis - cli - raw”将底层编码的字符串转换为中文。演示一下：

```bash
[root@localhost src]#./redis - cli
127.0.0.1:6379> GET className
"\xe8\xbd\xa4\xe4\xbb\xb6\xe5\xb7\xa5\xe7\xa8\x8b\xe7\x8f\xad"
[root@localhost src]#./redis - cli - raw
127.0.0.1:6379> GET className
软件工程1班
```

#### 3.1 Redis数据类型之字符串（String）命令

字符串类型是Redis中最基本的数据类型，它是二进制安全的，任何形式的字符串都可以存储，包括二进制数据、序列化后的数据、JSON化的对象，甚至是一张经Base64编码后的图片。String类型的键最大能存储512MB的数据。

Redis字符串类型的相关命令用于管理Redis的字符串值。下面以一张学生表为例，来讲解String类型的相关命令，如表3.1所示。

|stuName|stuID|age|sex|height|weight|birthday|className|
| ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
|刘河飞|20180001|22|男|171|75|1996 - 02 - 14|软件工程1班|
|赵雨梦|20181762|24|女|175|73|1994 - 04 - 23|网络工程1班|
|宋飞|20180023|23|男|168|67|1995 - 08 - 18|软件工程2班|
|陈慧|20181120|23|女|170|64|1995 - 03 - 15|信息管理1班|
|孙玉|20180097|22|女|166|63|1996 - 09 - 10|软件工程2班|


为了避免键值对出现覆盖的现象，我们统一在每个学生的键后面加上序号，如stuName - 1、stuID - 1。下面来看具体操作。

##### 3.1.1 设置键值对

1. **SET命令：设置键值对**

    - **命令格式**：SET key value [EX seconds] [PX milliseconds] [NX|XX]
    - 使用SET命令将字符串值value设置到key中。如果key中已经存在其他值，则在执行SET命令后，将会覆盖旧值，并且忽略类型。针对某个带有生存时间的key来说，当SET命令成功执行时，这个key上的生存时间会被清除。
    - **SET命令的可选参数如下**：
        - EX seconds：用于设置key的过期时间为多少秒（seconds）。其中，SET key value EX seconds等价于SETEX key seconds value。
        - PX milliseconds：用于设置key的过期时间为多少毫秒（milliseconds）。其中，SET key value PX milliseconds等价于PSETEX key milliseconds value。 
        - NX：表示当key不存在时，才对key进行设置操作。其中，SET key value NX等价于SETNX key value。 
        - XX：表示当key存在时，才对key进行设置操作。
    - **返回值**：如果SET命令设置成功，则会返回OK。如果设置了NX或XX，但因为条件不足而设置失败，则会返回空批量回复（NULL Bulk Reply）。
    - **使用SET命令将一条学生信息添加到数据库中，操作如下**：
```bash
127.0.0.1:6379> SET stuName - 1 '刘河飞'
OK
127.0.0.1:6379> SET stuID - 1 20180001
OK
127.0.0.1:6379> SET age - 1 22
OK
127.0.0.1:6379> SET sex - 1 '男'
OK
127.0.0.1:6379> SET height - 1 171
OK
127.0.0.1:6379> SET weight - 1 75
OK
127.0.0.1:6379> SET birthday - 1 1996 - 02 - 14
OK
127.0.0.1:6379> SET className - 1 '软件工程1班'
OK
```
2. **MSET命令：设置多个键值对**
    - **命令格式**：MSET key value [key value...]
    - 使用MSET命令同时设置多个键值对。如果某个key已经存在，那么MSET命令会用新值覆盖旧值。MSET命令是一个原子性操作，所有给定key都会在同一时间内被设置更新，不存在某些key被更新了而另一些key没有被更新的情况。
    - **返回值**：总是返回OK，因为MSET命令不可能设置失败。
    - **使用SET命令来逐个添加学生信息到数据库中会比较麻烦，下面使用MSET命令来一次性添加多个键值对，操作如下**：
```bash
127.0.0.1:6379> MSET stuName - 2 '赵雨梦' stuID - 2 20181762 age - 2 24 sex - 2 '女' height - 2 175 weight - 2 73 birthday - 2 1994 - 04 - 23 className - 2 '网络工程1班'
OK
127.0.0.1:6379> MSET stuName - 3 '宋飞' stuID - 3 20180023 age - 3 23 sex - 3 '男' height - 3 168 weight - 3 67 birthday - 3 1995 - 08 - 18 className - 3 '软件工程2班'
OK
127.0.0.1:6379> MSET stuName - 4 '陈慧' stuID - 4 20181120 age - 4 23 sex - 4 '女' height - 4 170 weight - 4 64 birthday - 4 1995 - 03 - 15 className - 4 '信息管理1班'
OK
127.0.0.1:6379> MSET stuName - 5 '孙玉' stuID - 5 20180097 age - 5 22 sex - 5 '女' height - 5 166 weight - 5 63 birthday - 5 1996 - 09 - 10 className - 5 '软件工程2班'
OK
```
3. **SETNX命令：设置不存在的键值对**
   
    - **命令格式**：SETNX key value
    - SETNX是set if not exists的缩写。如果key不存在，则设置值，当且仅当key不存在时。如果key已经存在，则SETNX什么也不做。
    - **返回值**：SETNX命令设置成功返回1，设置失败返回0。
    - **设置学院名称，操作如下**：
```bash
127.0.0.1:6379> SETNX collegeName '计算机学院'  #设置学院名称为“计算机学院”
1
127.0.0.1:6379> SETNX collegeName '计算机工程学院'  #更名为“计算机工程学院”
0
```
4. **MSETNX命令：设置多个不存在的键值对**
    - **命令格式**：MSETNX key value [key value...]
    - 使用MSETNX命令同时设置多个键值对，当且仅当所有给定key都不存在时设置。与MSET命令类似，如果有一个给定key已经存在，那么MSETNX命令也会拒绝执行所有给定key的设置操作。MSETNX命令是原子性的，它要么全部设置成功，要么全部设置失败。
    - **返回值**：所有key设置成功返回1；如果所有给定key都设置失败，则返回0。
    - **为学生分配多名专业课老师，操作如下**：
```bash
127.0.0.1:6379> MSETNX Chinese - teacher '郭涛' Math - teacher '杨艳' English - teacher '吴芳'  #为学生分配语文、数学、英语老师
1
127.0.0.1:6379> MSETNX Chinese - teacher '陈诚' Math - teacher '杨小艳'  #更换语文、数学老师
0
```
##### 3.1.2 获取键值对

1. **GET命令：获取键值对的值**
    - **命令格式**：GET key
    - 使用GET命令获取key中设置的字符串值。如果key中存储的值不是字符串类型的，则会返回一个错误，因为GET命令只能用于处理字符串的值；当key不存在时，返回nil。
    - **返回值**：当key存在时，返回key所对应的值；如果key不存在，则返回nil；如果key不是字符串类型的，则返回错误。
    - **使用GET命令查看学生的基本信息，操作如下**：
```bash
127.0.0.1:6379> GET stuName - 1
刘河飞
127.0.0.1:6379> GET stuID - 1
20180001
127.0.0.1:6379> GET age - 1
22
127.0.0.1:6379> GET sex - 1
男
127.0.0.1:6379> GET height - 1
171
127.0.0.1:6379> GET weight - 1
75
127.0.0.1:6379> GET birthday - 1
1996 - 02 - 14
127.0.0.1:6379> GET className - 1
软件工程1班
```
2. **MGET命令：获取多个键值对的值**
   
    - **命令格式**：MGET key [key...]
    - 使用MGET命令同时返回多个给定key的值，key之间使用空格隔开。如果在给定的key中有不存在的key，那么这个key返回的值为nil。
    - **返回值**：一个包含所有给定key的值的列表。
    - **使用MGET命令来获取多名学生的信息，操作如下**：
```bash
127.0.0.1:6379> MGET stuName - 1 stuID - 1 age - 1 sex - 1 height - 1 weight - 1 birthday - 1 className - 1 #获取第一名学生的信息
刘河飞
20180001
22
男
171
75
1996 - 02 - 12
软件工程1班
127.0.0.1:6379> MGET stuName - 1 stuName - 2 stuName - 3 stuName - 4 stuName - 5  #获取5名学生的姓名
刘河飞
赵雨梦
宋飞
陈慧
孙玉
```
3. **GETRANGE命令：获取键的子字符串值**
   
    - **命令格式**：GETRANGE key start end
    - 使用GETRANGE命令来获取key中字符串值从start开始到end结束的子字符串，下标从0开始（字符串截取）。start和end参数是整数，可以取负值。当取负值时，表示从字符串最后开始计数，-1表示最后一个字符，-2表示倒数第二个字符，以此类推。
    - **返回值**：返回截取的子字符串。
    - **操作如下**：
```bash
127.0.0.1:6379> SET motto - 2 '一个人只有全面回忆自己，才能认识真正的自己'  #设置学生2的座右铭
OK
127.0.0.1:6379> GETRANGE motto - 2 0 100  #截取座右铭的子字符串
一个人只有全面回忆自己，才能认识真正的自己
127.0.0.1:6379> GETRANGE motto - 2 -8 -1
自己
127.0.0.1:6379> GETRANGE motto - 2 0 -3
一个人只有全面回忆自己，才能认识真正的自
127.0.0.1:6379> GETRANGE motto - 2 0 -5
一个人只有全面回忆自己，才能认识真正的
```
##### 3.1.3 键值对的偏移量

1. **SETBIT命令：设置键的偏移量**
   
    - **命令格式**：SETBIT key offset value
    - 使用SETBIT命令对key所存储的字符串值设置或清除指定偏移量上的位（bit）。value参数值决定了位的设置或清除，value取值0或1。当key不存在时，自动生成一个新的字符串。这个字符串是动态的，它可以扩展，以确保将value保存在指定的偏移量上。当这个字符串扩展时，使用0来填充空白位置。offset参数必须是大于或等于0，并且小于2^32（bit映射被限制在512MB之内）的正整数。在默认情况下，bit初始化为0。
    - **返回值**：返回指定偏移量原来存储的位。
    - **设置学生1姓名及学院名的偏移量，操作如下**：
```bash
127.0.0.1:6379> SETBIT stuName - 1 6 1
0
127.0.0.1:6379> SETBIT stuName - 1 7 0
1
127.0.0.1:6379> SETBIT collegeName 100 0
1
```
2. **GETBIT命令：获取键的偏移量值**
   
    - **命令格式**：GETBIT key offset
    - 对key所存储的字符串值，使用GETBIT命令来获取指定偏移量上的位（bit）。当offset的值超过了字符串的最大长度，或者key不存在时，返回0。
    - **返回值**：返回字符串值指定偏移量上的位（bit）。
    - **获取学生1姓名及学院名的偏移量值，操作如下**：
```bash
127.0.0.1:6379> GETBIT stuName - 1 6
1
127.0.0.1:6379> GETBIT stuName - 1 7
0
127.0.0.1:6379> GETBIT collegeName 100
0
```
##### 3.1.4 设置键的生存时间
1. **SETEX命令：为键设置生存时间（秒）**
    - **命令格式**：SETEX key seconds value
    - 使用SETEX命令将value值设置到key中，并设置key的生存时间为多少秒（seconds）。

### 3.1.4 设置键的生存时间（续）

1. **SETEX命令：为键设置生存时间（秒）**

如果key已经存在，则SETEX命令将覆写旧值。

它等价于：
```
SET key value
EXPIRE key seconds
```
SETEX命令是一个原子性命令，它设置value与设置生存时间是在同一时间完成的。

返回值：设置成功时返回OK；当seconds参数不合法时，返回错误。

**为不存在的键（学校名）设置生存时间，操作如下**：

```bash
127.0.0.1:6379> SETEX schoolName 100 '清华大学' #设置学校名为“清华大学”，生存时间为100秒
OK
127.0.0.1:6379> GET schoolName #获取学校名
清华大学
127.0.0.1:6379> TTL schoolName #查看剩余生存时间，剩余55秒
55
127.0.0.1:6379> GET schoolName #55秒后查看，键值为空
```
2. **PSETEX命令：为键设置生存时间（毫秒）**

    - **命令格式**：PSETEX key milliseconds value
    - 使用PSETEX命令设置key的生存时间，以毫秒为单位。设置成功时返回OK。
    - **以毫秒的形式，设置键（学校地址）的生存时间，操作如下**：
```bash
127.0.0.1:6379> PSETEX school-address 30000 '北京' #设置学校地址为“北京”，生存时间为30000毫秒
OK
127.0.0.1:6379> GET school-address #获取学校地址
北京
127.0.0.1:6379> PTTL school-address #查看学校地址剩余多少生存时间（毫秒）
17360
127.0.0.1:6379> GET school-address
```
### 3.1.5 键值对的值操作

1. **SETRANGE命令：替换键的值**
   
    - **命令格式**：SETRANGE key offset value
    - 使用SETRANGE命令从指定的位置（offset）开始将key的值替换为新的字符串。比如旧值为hello world，执行命令SETRANGE 5 Redis，表示从第5个下标位置开始替换为新的字符串Redis，其结果为helloredis。如果有不存在的key，就当作空白字符串处理。SETRANGE命令会确保字符串足够长，以便将value设置在指定的偏移量上。
    - 如果给定key原来存储的字符串长度比偏移量小（比如，字符串只有4个字符长，但设置的offset是9），那么原字符和偏移量之间的空白将用零字节（Zerobytes，“\x00”）来填充。
    - 注意：offset的最大偏移量是2^29 - 1（536870911），因为Redis字符串的大小被限制在512MB以内。假如需要使用比这更大的空间，则可以使用多个key来存储。
    - **返回值**：返回执行SETRANGE命令之后的字符串的长度。
    - **替换学生2、学生3的座右铭，操作如下**：
```bash
127.0.0.1:6379> GET motto-2 #查看学生2的座右铭
一个人只有全面回忆自己，才能认识真正的自己
127.0.0.1:6379> SETRANGE motto-2 21 '美好过去' #替换学生2的座右铭，从偏移量21开始
63
127.0.0.1:6379> GET motto-2
一个人只有全面是美好过去，才能认识真正的自己
127.0.0.1:6379> EXISTS motto-3 #学生3的座右铭不存在（对一个不存在的键进行替换）
0
127.0.0.1:6379> SETRANGE motto-3 12 'hello world' #替换学生3的座右铭，从偏移量12开始
(integer) 23
127.0.0.1:6379> GET motto-3
"\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00hello world"
```
2. **GETSET命令：为键设置新值**
   
    - **命令格式**：GETSET key value
    - 使用GETSET命令将给定key的值设置为value，并返回key的旧值。当key存在但不是字符串类型时，将会返回错误。
    - **返回值**：返回给定key的旧值。如果key不存在，则返回nil；如果key存在但不是字符串类型的，则返回错误。
    - **先为学生1设置座右铭，再修改，操作如下**：
```bash
127.0.0.1:6379> EXISTS motto-1 #检查键motto-1是否存在
0
127.0.0.1:6379> GETSET motto-1 '没有存款，就是拼的理由' #没有旧值，返回空
没有存款，就是拼的理由
127.0.0.1:6379> GET motto-1 #获取值
没有存款，就是拼的理由
127.0.0.1:6379> GETSET motto-1 '拼个春夏秋冬，赢个无悔人生' #为键motto-1设置新值
没有存款，就是拼的理由
127.0.0.1:6379> GET motto-1
拼个春夏秋冬，赢个无悔人生
```
3. **APPEND命令：为键追加值**
   
    - **命令格式**：APPEND key value
    - 如果key存在且是字符串类型的，则将value值追加到key旧值的末尾。如果key不存在，则将key设置值为value。
    - **返回值**：返回追加value之后，key中字符串的长度。
    - **为学生1的座右铭追加新值，操作如下**：
```bash
127.0.0.1:6379> GET motto-1 #获取键motto-1的值
拼个春夏秋冬，赢个无悔人生
127.0.0.1:6379> APPEND motto-1 '，努力过，' #追加新值
50
127.0.0.1:6379> GET motto-1
拼个春夏秋冬，赢个无悔人生，努力过，
127.0.0.1:6379> APPEND motto-1 '此生无憾' #追加新值
62
127.0.0.1:6379> GET motto-1
拼个春夏秋冬，赢个无悔人生，努力过，此生无憾
```
### 3.1.6 键值对的计算
1. **BITCOUNT命令：计算比特位数量**
   
    - **命令格式**：BITCOUNT key [start] [end]
    - 使用BITCOUNT命令计算在给定的字符串中被设置为1的比特位数量。它有两个参数start和end。如果不设置这两个参数，则表示它会对整个字符串进行计数；如果指定了这两个参数值，则表示计数只在特定的位上进行。start和end参数都是整数值，可以取负数，比如，-1表示字符串的最后一个字节，-2表示字符串的倒数第二个字节，以此类推。如果被计数的key不存在，就会被当成空字符串来处理，其计数结果为0。
    - **返回值**：执行BITCOUNT命令之后，返回被设置为1的位的数量。
    - **计算学生3姓名的比特位数量，操作如下**：
```bash
127.0.0.1:6379> BITCOUNT stuName-3 #计算学生3姓名的比特位数量，为28
28
127.0.0.1:6379> BITCOUNT stuName-3 0 10 #计算学生3姓名在0~10之间的比特位数量
28
127.0.0.1:6379> BITCOUNT stuName-3 0 3
19
127.0.0.1:6379> SETBIT stuName-3 100 1 #设置学生3姓名在偏移量为100处的比特位
0
127.0.0.1:6379> BITCOUNT stuName-3 #28+1
29
```
2. **BITOP命令：对键进行位元运算**
   
    - **命令格式**：BITOP operation destkey key [key...]
    - 使用BITOP命令对一个或多个保存二进制位的字符串key进行位元运算，并将运算结果保存到destkey中。operation表示位元操作符，它可以是AND、OR、NOT、XOR这4种操作中的任意一种。具体操作如下：
        - BITOP AND destkey key [key...]：表示对一个或多个key求逻辑并，并将结果保存到destkey中。
        - BITOP OR destkey key [key...]：表示对一个或多个key求逻辑或，并将结果保存到destkey中。
        - BITOP NOT destkey key：表示对给定key求逻辑非，并将结果保存到destkey中。
        - BITOP XOR destkey key [key...]：表示对一个或多个key求逻辑异或，并将结果保存到destkey中。
    - 除NOT操作之外，其余的操作都可以接收一个或多个key作为参数。当使用BITOP命令来进行不同长度的字符串的位元运算时，较短的那个字符串所缺少的部分将会被看作0。空的key也被看作包含0的字符串序列。
    - **返回值**：返回保存到destkey中的字符串的长度，这个长度和输入key中最长的字符串的长度相等。
    - **对学生1和学生2的年龄进行位元操作，操作如下**：
```bash
127.0.0.1:6379> SETBIT age-1 0 1 #设置学生1年龄的比特位
1
127.0.0.1:6379> SETBIT age-2 3 1 #设置学生2年龄的比特位
1
127.0.0.1:6379> BITOP AND result age-1 age-2 #逻辑与运算
2
127.0.0.1:6379> BITOP OR result1 age-1 age-2 #逻辑或运算
2
127.0.0.1:6379> BITOP NOT result2 age-1 age-2 #逻辑非运算
ERR BITOP NOT must be called with a single source key.
127.0.0.1:6379> BITOP NOT result3 age-1
2
127.0.0.1:6379> BITOP NOT result4 age-2
2
127.0.0.1:6379> BITOP XOR results age-1 age-2 #逻辑异或运算
1
127.0.0.1:6379> GETBIT age-1 0 #获取偏移量为0的值
1
127.0.0.1:6379> BITCOUNT age-1 #统计比特位数量
7
127.0.0.1:6379> BITCOUNT age-2
6
```
3. **STRLEN命令：统计键的值的字符长度**
    - **命令格式**：STRLEN key
    - 使用命令STRLEN统计key的值的字符长度。当key存储的不是字符串时，返回一个 



   错误。当key不存在时，返回0。

**统计学生1和学生2的座右铭的长度，操作如下**：
```bash
127.0.0.1:6379> GET motto-1
拼个春夏秋冬，赢个无悔人生，努力过，此生无憾
127.0.0.1:6379> STRLEN motto-1
62
127.0.0.1:6379> GET motto-2
一个人只有全面美好过去，才能认识真正的自己
127.0.0.1:6379> STRLEN motto-2
63
```
### 3.1.7 键值对的值增量

1. **DECR命令：让键的值减1**
   
    - **命令格式**：DECR key
    - 使用DECR命令将key中存储的数字值减1。如果key不存在，则key的值先被初始化为0，再执行DECR操作减1。注意，这个命令只能对数字类型的数据进行操作（自减）。如果key对应的值包含错误类型，或者字符串类型的值不能表示为数字，则将返回一个错误。DECR操作的值限制在64位（bit）有符号数字表示范围之内。
    - **返回值**：返回执行DECR操作之后key的值。
    - **让学生2的年龄、身高、体重、生日都减1，操作如下**：
```bash
127.0.0.1:6379> MGET stuName-2 age-2 sex-2 height-2 weight-2 birthday-2 className-2
赵雨梦
24
女
175
73
1994-04-23
网络工程1班
127.0.0.1:6379> DECR age-2  #年龄减1
23
127.0.0.1:6379> DECR height-2  #身高减1
174
127.0.0.1:6379> DECR weight-2  #体重减1
72
127.0.0.1:6379> DECR birthday-2  #类型错误
ERR value is not an integer or out of range
```
2. **DECRBY命令：键的值减去减量值**
   
    - **命令格式**：DECRBY key decrement
    - 使用DECRBY命令将key所存储的值减去减量值decrement。如果key不存在，则key的值先被初始化为0，再执行DECRBY命令。如果DECRBY命令操作的值包含错误的类型，或者字符串类型的值不能表示为数字，则将返回一个错误。DECRBY操作的数值限制在64位（bit）有符号数字表示范围之内。
    - **返回值**：返回减去减量值decrement的key的新值。
    - **让学生3的学号、年龄、身高、体重都减去减量值5，操作如下**：
```bash
127.0.0.1:6379> MGET stuName-3 stuID-3 age-3 sex-3 height-3 weight-3 birthday-3 className-3
宋飞
20180023
23
男
168
67
1995-08-18
软件工程2班
127.0.0.1:6379> DECRBY stuID-3 5
20180018
127.0.0.1:6379> DECRBY age-3 5
18
127.0.0.1:6379> DECRBY height-3 5
163
127.0.0.1:6379> DECRBY weight-3 5
62
```
3. **INCR命令：让键的值加1**
   
    - **命令格式**：INCR key
    - 使用INCR命令将key中存储的数字值加1。如果key不存在，则key的值先被初始化为0，再执行INCR操作加1。注意，这个命令只能对数字类型的数据进行操作（自增）。如果INCR操作的值包含错误的类型，或者字符串类型的值不能表示为数字，则将返回一个错误。INCR操作的值限制在64位（bit）有符号数字表示范围之内。
    - **返回值**：返回执行INCR命令之后key的新值。
    - **让学生4的学号、年龄、身高、体重都加1，操作如下**：
```bash
127.0.0.1:6379> MGET stuName-4 stuID-4 age-4 sex-4 height-4 weight-4 birthday-4 className-4
陈慧
20181120
23
女
170
64
1995-03-15
信息管理1班
127.0.0.1:6379> INCR stuID-4
20181121
127.0.0.1:6379> INCR age-4
24
127.0.0.1:6379> INCR height-4
171
127.0.0.1:6379> INCR weight-4
65
```
4. **INCRBY命令：让键的值加上增量值**
   
    - **命令格式**：INCRBY key increment
    - 使用INCRBY命令将key所存储的值加上增量值increment。如果key不存在，则key的值先被初始化为0，再执行INCRBY命令。如果INCRBY命令操作的值包含错误的类型，或者字符串类型的值不能表示为数字，则将返回一个错误。INCRBY操作的值限制在64位（bit）有符号数字表示范围之内。
    - **返回值**：返回加上增量值increment之后的key的新值。
    - **让学生5的学号、年龄、身高、体重都加上增量值3，操作如下**：
```bash
127.0.0.1:6379> MGET stuName-5 stuID-5 age-5 sex-5 height-5 weight-5 birthday-5 className-5
孙玉
20180097
22
女
166
63
1996-09-10
软件工程2班
127.0.0.1:6379> INCRBY stuID-5 3
20180100
127.0.0.1:6379> INCRBY age-5 3
25
127.0.0.1:6379> INCRBY height-5 3
169
127.0.0.1:6379> INCRBY weight-5 3
66
```
5. **INCRBYFLOAT命令：让键的值加上浮点数增量值**
   
    - **命令格式**：INCRBYFLOAT key increment
    - 使用INCRBYFLOAT命令将key所存储的值加上浮点数增量值increment。如果key不存在，则key的值先被初始化为0，再执行INCRBYFLOAT命令。如果INCRBYFLOAT命令执行成功，那么key的值会被更新为命令执行后的新值，并且新值以字符串的形式返回给调用者。产生的新值和浮点数增量值increment都可以使用像3.0e9、4e7、8e - 3这样的指数符号来表示。
    - 在执行INCRBYFLOAT命令之后，产生的值会以相同的形式存储，都是由一个数字、一个可选的小数点、一个任意位数的小数部分组成的。小数部分末尾的0会被忽略掉。在有特定需要时，浮点数也会转化为整数，比如，2.0会被保存为2。如果INCRBYFLOAT操作的key的值不是字符串类型的，或者key所对应的值、给定的浮点数增量值increment不能表示为双精度浮点数，则
  
   在执行INCRBYFLOAT命令之后，产生的值会以相同的形式存储，都是由一个数字、一个可选的小数点、一个任意位数的小数部分组成的。小数部分末尾的0会被忽略掉。在有特定需要时，浮点数也会转化为整数，比如，2.0会被保存为2。如果INCRBYFLOAT操作的key的值不是字符串类型的，或者key所对应的值、给定的浮点数增量值increment不能表示为双精度浮点数，则会返回一个错误。

**返回值**：返回执行INCRBYFLOAT命令之后key的新值。

**注意**：在执行INCRBYFLOAT命令之后，无论产生的浮点数的实际精度有多长，计算结果最多也只能保留小数点后17位。

**对圆周率PI加上浮点数增量值，操作如下**：

```bash
127.0.0.1:6379> SET PI 3.1415
OK
127.0.0.1:6379> INCRBYFLOAT PI 0.002
3.1435
127.0.0.1:6379> SET PI-1 31415e-4
OK
127.0.0.1:6379> GET PI-1
31415e-4
127.0.0.1:6379> INCRBYFLOAT PI-1 0.0001
3.1416
127.0.0.1:6379> SET PI-2 314
OK
127.0.0.1:6379> INCRBYFLOAT PI-2 0.15
314.14999999999999999
127.0.0.1:6379> SET PI-3 314.0
OK
127.0.0.1:6379> INCRBYFLOAT PI-3 1.00000000000000000
315
```
### 3.2 Redis数据类型之哈希（Hash）命令

Redis的Hash类型是一个String类型的域（field）和值（value）的映射表，Hash数据类型常常用来存储对象信息。在Redis中，每个哈希表可以存储2^32 - 1个键值对，也就是40多亿个数据。


下面仍以学生表3.1为例，切换到1号数据库，讲解Redis数据库的哈希命令。

#### 3.2.1 设置哈希表域的值

1. **HSET命令：为哈希表的域设值**
    - **命令格式**：HSET key field value
    - 使用HSET命令将哈希表key中的field的值设置为value。当这个key不存在时，将会创建一个新的哈希表并进行HSET操作。如果field已经存在于哈希表中，那么新值将会覆盖旧值。
    - **返回值**：在哈希表中，如果field是一个新建域，并且HSET操作成功了，则将会返回1；如果哈希表中已经存在field，那么在新值覆盖旧值后，将会返回0。
    - **添加学生1到哈希表student1中，操作如下**：
```bash
127.0.0.1:6379[1]> HSET student1 stuName '刘河飞'
1
127.0.0.1:6379[1]> HSET student1 stuID 20180001
1
127.0.0.1:6379[1]> HSET student1 age 22
1
127.0.0.1:6379[1]> HSET student1 sex '男'
1
127.0.0.1:6379[1]> HSET student1 height 171
1
127.0.0.1:6379[1]> HSET student1 weight 75
1
127.0.0.1:6379[1]> HSET student1 birthday 1996-02-14
1
127.0.0.1:6379[1]> HSET student1 className '软件工程1班'
1
```

2. **HSETNX命令：为哈希表不存在的域设值**
   
    - **命令格式**：HSETNX key field value
    - 使用HSETNX命令当且仅当域field不存在时，将哈希表key中的field的值设置为value。如果field已经存在，那么HSETNX命令将会执行无效。如果key不存在，则会首先创建一个新key，然后执行HSETNX命令。
    - **返回值**：设置成功则返回1；如果field已经存在，设置失败，则将会返回0。
    - **为学生1设置座右铭，操作如下**：
```bash
127.0.0.1:6379[1]> HSETNX student1 motto '拼个春夏秋冬，赢个无悔人生'
1
127.0.0.1:6379[1]> HSETNX student1 motto '拼个春夏秋冬，赢个无悔人生'
0
```
3. **HMSET命令：设置多个域和值到哈希表中**
   
    - **命令格式**：HMSET key field value [field value ...]
    - HMSET命令用于将一个或多个域 - 值（field - value）对设置到哈希表key中。执行该命令后，将会覆盖哈希表key中原有的域。当key不存在时，会创建一个空的哈希表并执行HMSET操作。
    - **返回值**：当HMSET命令执行成功时，返回OK；当key不是哈希类型时，直接返回错误。
    - **分别添加多名学生信息到哈希表中，操作如下**：
```bash
127.0.0.1:6379[1]> HMSET student2 stuName '赵雨梦' stuID 20181762 age 24 sex '女' height 175 weight 73 birthday '1994-04-23' className '网络工程1班'
OK
127.0.0.1:6379[1]> HMSET student3 stuName '宋飞' stuID 20180023 age 23 sex '男' height 168 weight 67 birthday '1995-08-18' className '软件工程2班'
OK
127.0.0.1:6379[1]> HMSET student4 stuName '陈慧' stuID 20181120 age 23 sex '女' height 170 weight 64 birthday '1995-03-15' className '信息管理1班'
OK
127.0.0.1:6379[1]> HMSET student5 stuName '孙玉' stuID 20180097 age 22 sex '女' height 166 weight 63 birthday '1996-09-10' className '软件工程2班'
OK
```
#### 3.2.2 获取哈希表中的域和值
1. **HGET命令：获取哈希表中域的值**
    - **命令格式**：HGET key field
    - 使用HGET命令获取哈希表key中field的值。
    - **返回值**：返回field的值；如果这个key不存在，或者field不存在，则将返回nil。
    - **获取哈希表student1中域的值，操作如下**：
```bash
127.0.0.1:6379[1]> HGET student1 stuName
刘河飞
127.0.0.1:6379[1]> HGET student1 stuID
20180001
127.0.0.1:6379[1]> HGET student1 age
22
127.0.0.1:6379[1]> HGET student1 sex
男
127.0.0.1:6379[1]> HGET student1 height
171
127.0.0.1:6379[1]> HGET student1 weight
75
127.0.0.1:6379[1]> HGET student1 birthday
1996-02-14
127.0.0.1:6379[1]> HGET student1 className
软件工程1班
```
2. **HGETALL命令：获取哈希表中所有的域和值**
    - **命令格式**：HGETALL key
    - 使用HGETALL命令获取哈希表key中所有的field和value。
    - **返回值**：执行该命令后，将会以列表的形式返回哈希表中的域（field）和值（value）。此时返回值的长度是哈希表长度的两倍。如果这个key不存在，则将返回空列表。
    - **获取哈希表student2中所有的域和值，操作如下**：
```bash
127.0.0.1:6379[1]> HGETALL student2
stuName
赵雨梦
stuID
20181762
age
24
sex
女
height
175
weight
73
birthday
1994-04-23
className
网络工程1班
```
3. **HMGET命令：获取多个域的值**
    - **命令格式**：HMGET key field [field ...]
    - HMGET命令用于获取哈希表key中一个或多个field的值。如果哈希表key中不存在这个field，则返回nil。而如果key不存在，则将会被当作一个空哈希表来处理，也会返回nil。
    - **返回值**：执行HMGET命令后，将会返回一个包含多个指定域（field）的关联值的表，表中值的顺序与给定域参数的请求顺序保持一致。
    - **获取学生3的姓名、学号、年龄、生日及班级，操作如下**：
```bash
127.0.0.1:6379[1]> HMGET student3 stuName stuID age birthday className
宋飞
20180023
23
1995-08-18
软件工程2班
```
4. **HKEYS命令：获取哈希表中的所有域**
    - **命令格式**：HKEYS key
    - HKEYS命令用于获取哈希表key中的所有域（field）。
    - **返回值**：执行该命令后，将会返回包含这个哈希表key中的所有域的表。当key不存在时，返回一个空表。
    - **获取哈希表student1中的所有域，操作如下**：
```bash
127.0.0.1:6379[1]> HKEYS student1
stuName
stuID
age
sex
height
weight
birthday
className
motto
```
5. **HVALS命令：获取哈希表中所有域的值**
    - **命令格式**：HVALS key
    - HVALS命令用于返回哈希表key中所有域的值。
    - **返回值**：返回一个包含哈希表key中所有域的值的表。当key不存在时，返回一个空表。
    - **获取哈希表student1中所有域的值，操作如下**：
```bash
127.0.0.1:6379[1]> HVALS student1
刘河飞
20180001
22
男
171
75
1996-02-14
软件工程1班
拼个春夏秋冬，赢个无悔人生
```
#### 3.2.3 哈希表统计
1. **HLEN命令：统计哈希表中域的数量**
    - **命令格式**：HLEN key
    - HLEN命令用于统计哈希表key中域的数量。
    - **返回值**：返回哈希表key中域的数量，是一个数值。如果key不存在，则返回0，表示一个域也没有。
    - **统计哈希表student1中域的数量，操作如下**：
```bash
127.0.0.1:6379[1]> HKEYS student1
stuName
stuID
age
sex
height
weight
birthday
className
motto
127.0.0.1:6379[1]> HLEN student1
9
``` 
