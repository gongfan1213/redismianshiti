### 第4章 Redis必备命令
第3章讲解了Redis的5种数据类型的相关命令，本章讲解Redis的其他相关命令，具体包括Redis的键命令、HyperLogLog命令、脚本命令、连接命令、服务器命令等。通过对这些命令相关作用的讲解，结合实际操作，让读者更加熟练地操作Redis。

#### 4.1 键（key）命令
Redis的键命令主要用于管理Redis的键，如删除键、查询键、修改键及设置某个键等。我们以一张学生信息表为例，切换到5号数据库，来讲解Redis的键命令。学生信息表如表4.1所示。

**表4.1 学生信息表**
| name | password | age | sex | score | mobile | address | className |
| ---- | -------- | --- | --- | ----- | ------ | ------- | --------- |
| 刘河飞 | 123456 | 24 | 男 | 98 | 18293314444 | 深圳 | 软件工程1班 |
| 赵小雨 | 453762 | 23 | 女 | 87 | 13542130000 | 北京 | 金融1班 |
| 武恬 | 549871 | 22 | 女 | 94 | 13678543229 | 武汉 | 英语3班 |
| 周明 | 765892 | 25 | 男 | 63 | 13577889000 | 广州 | 建筑工程2班 |
| 王丽 | 842132 | 23 | 女 | 77 | 18397777555 | 上海 | 日语1班 |

##### 4.1.1 查询键
1. **EXISTS命令：判断键是否存在**
    - **命令格式**：EXISTS key
    - EXISTS命令用于判断指定的key是否存在。
    - **返回值**：EXISTS命令成功执行后，如果key存在，则返回1；如果key不存在，则返回0。
    - **操作示例**：判断学生的name键、password键、age键、sex键、score键、mobile键是否存在。
      - 127.0.0.1:6379> SELECT 5 #切换到5号数据库
      - 127.0.0.1:6379[5]> EXISTS name 0 #返回0，表示键不存在
      - 127.0.0.1:6379[5]> EXISTS password age sex score 0
      - 127.0.0.1:6379[5]> EXISTS mobile 0

2. **KEYS命令：查找键**
    - **命令格式**：KEYS pattern
    - KEYS命令用于按照指定的模式（pattern）查找所有的key。参数pattern类似于正则表达式。
    - **示例**：
      - KEYS * ：表示匹配radis、redis、rxd is等。
      - KEYS r?dis：表示匹配rdis、redis、ree dis等。
      - KEYS r*dis：表示匹配radis、redis等。
      - KEYS r[ae]dis：表示匹配radis和redis，但是不会匹配ridis。
    - 遇到特殊符号需要使用“\”隔开（转义）。
    - **返回值**：KEYS命令成功执行后，返回一个符合模式（pattern）的key列表。
    - **操作示例**：查找5号数据库中的键。
      - 127.0.0.1:6379> SELECT 5 #切换到5号数据库
      - 127.0.0.1:6379[5]> keys * #查找数据库中的所有键，为空，表示数据库中没有任何键
      - 127.0.0.1:6379[5]> SET red '红色' #设置多个键值对
      - 127.0.0.1:6379[5]> SET redis '数据库'
      - 127.0.0.1:6379[5]> SET reduce '减少'
      - 127.0.0.1:6379[5]> SET read '阅读'
      - 127.0.0.1:6379[5]> SET really '事实上'
      - 127.0.0.1:6379[5]> KEYS red* #查找以“red”开头的所有键
        - red
        - redis
        - reduce
      - 127.0.0.1:6379[5]> KEYS re* #查找以“re”开头的所有键
        - really
        - read
        - red
        - redis
        - reduce
      - 127.0.0.1:6379[5]> KEYS * #再次查找5号数据库中的键
        - really
        - read
        - red
        - redis
        - reduce

3. **OBJECT命令：查看键的对象**
    - **命令格式**：OBJECT subcommand [arguments [arguments]]
    - OBJECT命令用于从内部查看给定key的Redis对象。该命令通常用在除错或者为了节省空间而对key使用特殊编码的情况下。如果要用Redis来实现与缓存相关的功能，则可以使用OBJECT命令来决定是否清除key。
    - **OBJECT命令有如下子命令**：
      - OBJECT REFCOUNT key：用于返回给定key引用所存储的值的次数，多用于除错。
      - OBJECT ENCODING key：用于返回给定key所存储的值所使用的底层数据结构。
      - OBJECT IDLETIME key：用于返回给定key自存储以来的空闲时间，以秒为单位。
    - **Redis对象的编码格式**：
      - 针对字符串可以被编码为raw（一个字符串）或int（Redis会将字符串表示的64位有符号整数编码为整数来存储，以此来节约内存）。
      - 针对列表可以被编码为ziplist或linkedlist。ziplist是压缩列表，用来表示占用空间较小的列表。
      - 针对集合可以被编码为intset或hashtable。intset是只存储数字的小集合的特殊表示。
      - 针对哈希表可以被编码为zipmap或hashtable。zipmap是小哈希表的特殊表示。
      - 针对有序集合可以被编码为ziplist或skiplist。ziplist主要用于表示小的有序集合；而skiplist可以表示任意大小的有序集合。
    - **返回值**：OBJECT命令的子命令REFCOUNT和IDLETIME会返回数字，而ENCODING会返回相对应的编码类型。
    - **操作示例**：查看学生name键的对象信息。
      - 127.0.0.1:6379[5]> SET name '刘河飞' #设置学生的姓名
      - 127.0.0.1:6379[5]> OBJECT REFCOUNT name 1 #返回name键引用所存储的值的次数
      - 127.0.0.1:6379[5]> OBJECT IDLETIME name 35 #返回name键的空闲时间
      - 127.0.0.1:6379[5]> GET name 刘河飞
      - 127.0.0.1:6379[5]> OBJECT IDLETIME name 25
      - 127.0.0.1:6379[5]> OBJECT ENCODING name embstr #返回name键所存储的值所使用的底层数据结构

4. **RANDOMKEY命令：随机返回一个键**
    - **命令格式**：RANDOMKEY
    - RANDOMKEY命令用于随机返回当前数据库中的一个key，并且不会删除这个key。
    - **返回值**：如果这个数据库不为空，则将会返回一个随机key；如果这个数据库为空，则返回nil。
    - **操作示例**：随机返回5号数据库中的一个键。
      - 127.0.0.1:6379[5]> KEYS * #查看数据库中的所有键
        - really
        - read
        - name
        - red
        - redis
        - reduce
      - 127.0.0.1:6379[5]> RANDOMKEY #随机返回一个键
        - really
      - 127.0.0.1:6379[5]> RANDOMKEY 3 ERR wrong number of arguments for 'randomkey' command
      - 127.0.0.1:6379[5]> RANDOMKEY read
      - 127.0.0.1:6379[5]> RANDOMKEY red

##### 4.1.2 修改键
1. **RENAME命令：修改键的名称**
    - **命令格式**：RENAME key newkey
    - RENAME命令用于修改key的名称，将key的名称修改为newkey。如果这个key不存在，则返回一个错误。如果newkey已经存在，则RENAME命令执行后将会覆盖旧值。
    - **返回值**：RENAME命令成功执行后，返回OK，表示key的名称修改成功；当执行失败时，返回一个错误。
    - **操作示例**：修改5号数据库中学生name键的名称。
      - 127.0.0.1:6379[5]> GET name #获取name键的值
        - 刘河飞
      - 127.0.0.1:6379[5]> RENAME name stuName #修改name键的名称为“stuName”
        - OK
      - 127.0.0.1:6379[5]> GET name
      - 127.0.0.1:6379[5]> GET stuName
        - 刘河飞
      - 127.0.0.1:6379[5]> RENAME stuName username #再次修改stuName键的名称为“username”
        - OK
      - 127.0.0.1:6379[5]> GET username
        - 刘河飞
      - 127.0.0.1:6379[5]> EXISTS password 0
      - 127.0.0.1:6379[5]> RENAME password userPassword #修改一个不存在的键的名称，将会报错
        - ERR no such key

2. **RENAMENX命令：修改键的名称**
    - **命令格式**：RENAMENX key newkey
    - RENAMENX命令用于修改key的名称，将key的名称修改为newkey，当且仅当newkey不存在时才能修改。如果key不存在，则返回一个错误。
    - **返回值**：RENAMENX命令成功执行后，返回1，表示key的名称修改成功；如果newkey已经存在，则返回0。
    - **操作示例**：修改5号数据库中学生name键、className键的名称。
      - 127.0.0.1:6379[5]> SET className '软件工程1班' #设置className键值对
        - OK
      - 127.0.0.1:6379[5]> GET className
        - 软件工程1班
      - 127.0.0.1:6379[5]> RENAMENX className class - name #修改className键的名称为“class - name”
        - 1
      - 127.0.0.1:6379[5]> GET className
      - 127.0.0.1:6379[5]> GET class - name
        - 软件工程1班
      - 127.0.0.1:6379[5]> RENAMENX class - name class - name - 1 #再次修改为“class - name - 1”
        - 1
      - 127.0.0.1:6379[5]> GET class - name
      - 127.0.0.1:6379[5]> EXISTS mobile 0
      - 127.0.0.1:6379[5]> RENAMENX mobile phone #修改一个不存在的键的名称，将会报错
        - ERR no such key
      - 127.0.0.1:6379[5]> GET name
        - OK
      - 127.0.0.1:6379[5]> SET name '赵小雨'
      - 127.0.0.1:6379[5]> KEYS *
        - really
        - class - name - 1
        - read
        - name
        - userName
        - red
        - redis
        - reduce
      - 127.0.0.1:6379[5]> RENAMENX name username #将name键的名称修改为一个已经存在的键名称，将会失败
        - 0
      - 127.0.0.1:6379[5]> RENAMENX name girl - name
        - 1
      - 127.0.0.1:6379[5]> GET girl - name
        - 赵小雨

##### 4.1.3 键的序列化
1. **DUMP命令：序列化键**
    - **命令格式**：DUMP key
    - DUMP命令用于序列化给定的key，并返回被序列化的值。反之，我们可以使用RESTORE命令来反序列化这个key。
    - **序列化值的特点**：
      - 这个值具有64位的校验和，用于检测错误。RESTORE命令在进行反序列化之前，会先检查校验和。
      - 这个值的编码格式和RDB文件的编码格式保持一致。
      - RDB版本会被编码在序列化值中。如果Redis的版本不同，那么这个RDB文件会存在不兼容，Redis也就无法对这个值进行反序列化。
      - 这个序列化的值中没有生存时间信息。
    - **返回值**：如果key不存在，则返回nil；如果key存在，则DUMP命令成功执行后将会返回这个序列化的值。
    - **操作示例**：对学生的班级名className进行序列化。
      - 127.0.0.1:6379[5]> GET className #获取className键的值，为空
        - (nil)
      - 127.0.0.1:6379[5]> DUMP className #序列化一个空值的键，返回空
        - (nil)
      - 127.0.0.1:6379[5]> SET className '软件工程1班' #为className键设置值
        - OK
      - 127.0.0.1:6379[5]> GET className #获取className键的值
        - "软件工程1班"
      - 127.0.0.1:6379[5]> DUMP className #序列化className键
        - "\xe8\xbd\xaf\xe4\xbb\xb6\xe5\xb7\xa5\xe7\xa8\x8b1\xe7\x8f\xad"

2. **RESTORE命令：对序列化值进行反序列化**
    - **命令格式**：RESTORE key ttl serialized - value [REPLACE]
    - RESTORE命令用于将一个给定的序列化值反序列化，并为它关联给定的key。参数ttl用于为key设置生存时间，单位为毫秒。如果参数ttl的值为0，则不设置生存时间。
    - RESTORE命令在执行反序列化操作之前，会先对序列化的RDB版本和数据校验和进行检查。如果RDB版本不相同或者数据不完整，那么反序列化会失败，RESTORE拒绝进行反序列化，并返回一个错误。
    - 如果key已经存在，并且给定了REPLACE参数，那么使用反序列化得出的值来替换key的旧值；但是如果key已经存在，而没有设置REPLACE参数，则将会返回一个错误。
    - **返回值**：当RESTORE命令执行反序列化操作成功时，返回OK；失败时，返回一个错误。
    - **操作示例**：对前面序列化的className键进行反序列化。
      - 127.0.0.1:6379[5]> DUMP className "\x00\x10\xe8\xbd\xaf\xe4\xbb\xb6\xe5\xb7\xa5\xe7\xa8\x8b1\xe7\x8f\xad\xb0\x00\xdb\xe7\xd5\xed\xdd\x85\xc9\xd8"
      - 127.0.0.1:6379[5]> RESTORE className 0 "\x00\x10\xe8\xbd\xaf\xe4\xbb\xb6\xe5\xb7\xa5\xe7\xa8\x8b1\xe7\x8f\xad\xb0\x00\xdb\xe7\xd5\xed\xdd\x85\xc9\xd8" (error) BUSYKEY Target key name already exists.
      - 127.0.0.1:6379[5]> RESTORE className 0 "\x00\x10\xe8\xbd\xaf\xe4\xbb\xb6\xe5\xb7\xa5\xe7\xa8\x8b1\xe7\x8f\xad\xb0\x00\xdb\xe7\xd5\xed\xdd\x85
     
    以下是对剩余内容的提取：

### 第4章 Redis必备命令
#### 4.1 键（key）命令
##### 4.1.3 键的序列化（续）
2. **RESTORE命令：对序列化值进行反序列化**
    - **操作示例（续）**：
      - 127.0.0.1:6379[5]> RESTORE className 0 "\x00\x10\xe8\xbd\xaf\xe4\xbb\xb6\xe5\xb7\xa5\xe7\xa8\x8b1\xe7\x8f\xad\xb0\x00\xdb\xe7\xd5\xed\xdd\x85\xc9\xd8" REPLACE
        - OK
      - 127.0.0.1:6379[5]> GET className
        - "\xe8\xbd\xaf\xe4\xbb\xb6\xe5\xb7\xa5\xe7\xa8\x8b1\xe7\x8f\xad"

##### 4.1.4 键的生存时间
1. **PTTL命令：获取键的生存时间（毫秒）**
    - **命令格式**：PTTL key
    - PTTL命令用于以毫秒的形式返回key的剩余生存时间，与TTL命令相似。
    - **返回值**：PTTL命令成功执行后，会返回key的剩余生存时间，单位为毫秒。如果key不存在，则返回 - 2；如果key存在，但是并没有设置生存时间，则返回 - 1。
    - **操作示例**：获取学生name键、age键的生存时间。
      - 127.0.0.1:6379[5]> PTTL name
        - (integer) - 2 # - 2表示name键不存在
      - 127.0.0.1:6379[5]> PTTL age
        - (integer) - 1 # - 1表示age键存在，但是没有设置生存时间

2. **TTL命令：获取键的生存时间（秒）**
    - **命令格式**：TTL key
    - TTL命令用于返回key的剩余生存时间，以秒为单位。
    - **返回值**：TTL命令成功执行后，会返回key的剩余生存时间，单位为秒。如果key不存在，则返回 - 2；如果key存在，但是并没有设置生存时间，则返回 - 1。
    - **操作示例**：获取学生name键、age键的生存时间。
      - 127.0.0.1:6379[5]> TTL name
        - (integer) - 2 # - 2表示name键不存在
      - 127.0.0.1:6379[5]> TTL age
        - (integer) - 1 # - 1表示age键存在，但是没有设置生存时间

3. **EXPIRE命令：设置键的生存时间（秒）**
    - **命令格式**：EXPIRE key seconds
    - EXPIRE命令用于设置key的生命周期（生存时间）。当key的生存时间为0（过期）时，这个key将会被删除。
    - 如果想删除这个key的生存时间，则可以使用DEL命令连同这个key一起删除。
    - 如果想修改这个带有生存时间的key的值，则可以使用SET或GETSET命令来实现。SET或GETSET命令仅仅用于修改这个key的值，它的生存时间不会被修改。
    - 如果使用RENAME命令来修改一个key的名称，那么改名后的key的生存时间和改名前的key的生存时间一样。
    - 如果只想删除某个key的生存时间，而不想删除这个key，则可以使用PERSIST命令来实现让这个key成为一个持久的key。
    - 还可以使用EXPIRE命令来修改一个已经带有生存时间的key的生存时间，旧的生存时间会被新的生存时间替换（更新生存时间）。
    - **返回值**：当key存在时，EXPIRE命令成功执行后，返回1；如果key不存在，或者不能为key设置生存时间（Redis版本过低），则返回0。
    - **操作示例**：以秒的形式设置学生name键、age键的生存时间。
      - 127.0.0.1:6379[5]> GET name #获取name键的值，返回空，说明name键不存在
        - (nil)
      - 127.0.0.1:6379[5]> GET age
        - "23"
      - 127.0.0.1:6379[5]> EXPIRE name 60 #为name键设置生存时间为60秒
        - (integer) 0 #返回0，说明name键不存在或不能为name键设置生存时间
      - 127.0.0.1:6379[5]> EXPIRE age 60 #为age键设置生存时间为60秒
        - (integer) 1
      - 127.0.0.1:6379[5]> TTL name
        - (integer) - 2
      - 127.0.0.1:6379[5]> TTL age
        - (integer) 44
      - 127.0.0.1:6379[5]> PTTL age
        - (integer) 29827
      - 127.0.0.1:6379[5]> GET age #再次获取age键的值为空，说明生存时间已过，age键被删除了
        - (nil)

4. **PEXPIRE命令：设置键的生存时间（毫秒）**
    - **命令格式**：PEXPIRE key milliseconds
    - PEXPIRE命令用于以毫秒的形式设置key的生存时间，与EXPIRE命令相似。
    - **返回值**：当PEXPIRE命令为key设置生存时间成功时，返回1；如果key不存在，或者设置失败，则返回0。
    - **操作示例**：以毫秒的形式设置学生score键的生存时间。
      - 127.0.0.1:6379[5]> SET score 98 #设置score键的值为98
        - OK
      - 127.0.0.1:6379[5]> PTTL score #获取score键的生存时间
        - (integer) - 1 # - 1表示score键存在，但是没有设置生存时间
      - 127.0.0.1:6379[5]> PEXPIRE score 35000 #设置score键的生存时间为35000毫秒
        - (integer) 1
      - 127.0.0.1:6379[5]> PTTL score #获取score键的剩余生存时间为30232毫秒
        - (integer) 30232
      - 127.0.0.1:6379[5]> TTL score #获取score键的剩余生存时间为16秒
        - (integer) 16
      - 127.0.0.1:6379[5]> GET score
        - (nil)

5. **EXPIREAT命令：设置键的生存UNIX时间戳（秒）**
    - **命令格式**：EXPIREAT key timestamp
    - EXPIREAT命令用于为key设置生存时间，参数timestamp是UNIX时间戳。该命令的用法与EXPIRE命令的用法类似。
    - **返回值**：如果生存时间设置成功，则返回1；当key不存在，或者不能为key设置生存时间时，返回0。
    - **操作示例**：以秒为单位设置学生name键的生存UNIX时间戳。
      - 127.0.0.1:6379[5]> SET name '王丽'
        - OK
      - 127.0.0.1:6379[5]> TTL name
        - (integer) - 1
      - 127.0.0.1:6379[5]> EXPIREAT name 12629762426 #设置name键的生存UNIX时间戳
        - (integer) 1
      - 127.0.0.1:6379[5]> TTL name
        - (integer) 11087592233
      - 127.0.0.1:6379[5]> TTL name
        - (integer) 11087592213
      - 127.0.0.1:6379[5]> GET name
        - "\xe7\x8e\x8b\xe4\xbe\x8b"

6. **PEXPIREAT命令：设置键的生存UNIX时间戳（毫秒）**
    - **命令格式**：PEXPIREAT key milliseconds - timestamp
    - PEXPIREAT命令用于以毫秒为单位设置key的过期UNIX时间戳，与EXPIREAT命令相似。
    - **返回值**：当PEXPIREAT命令为key设置生存时间成功时，返回1；如果key不存在，或者不能为key设置生存时间，则返回0。
    - **操作示例**：以毫秒为单位设置学生mobile键的生存UNIX时间戳。
      - 127.0.0.1:6379[5]> SET mobile 18293314444
        - OK
      - 127.0.0.1:6379[5]> PEXPIREAT mobile 1500000000000 #设置mobile键的生存UNIX时间戳
        - (integer) 1
      - 127.0.0.1:6379[5]> PTTL mobile
        - (integer) - 2 #设置的生存UNIX时间戳时间过短
      - 127.0.0.1:6379[5]> SET mobile 18293314444
        - OK
      - 127.0.0.1:6379[5]> PEXPIREAT mobile 1500000000000000
        - (integer) 1
      - 127.0.0.1:6379[5]> PTTL mobile
        - (integer) 1498457829364592
      - 127.0.0.1:6379[5]> GET mobile
        - "18293314444"

##### 4.1.5 键值对操作
1. **MIGRATE命令：转移键值对到远程目标数据库**
    - **命令格式**：MIGRATE host port key destination - db timeout [COPY] [REPLACE]
    - MIGRATE命令用于将key原子性地从当前数据库转移（复制）到指定的目标数据库中，一旦转移成功，key就会出现在目标数据库中，当前数据库中的key就会被删除。
    - MIGRATE命令是原子操作，它在执行的时候会阻塞进行转移的两个数据库，直到转移成功，或转移失败，又或者出现等待超时。
    - **MIGRATE命令的实现原理**：在当前数据库（源数据库）中，对给定的key执行DUMP命令，将它序列化后，转移到目标数据库中，目标数据库再使用RESTORE命令对数据进行反序列化，将反序列化后的结果保存到数据库中；源数据库就好像目标数据库的客户端一样，只要遇到RESTORE命令返回OK，就会调用DEL命令删除自己数据库中的key。
    - **参数timeout**：用于设置当前数据库与目标数据库进行转移时的最大时间间隔，单位为毫秒。当转移的时间超过了timeout时，就会报请求超时。MIGRATE命令需要在timeout时间范围内完成I/O操作。如果在转移数据的过程中发生了I/O操作，或者达到了超时时间，那么该命令将会终止执行，并返回一个特殊的错误：IOERR。出现IOERR错误有两种情况：
      - 源数据库和目标数据库中可能同时存在这个key。
      - key也可能只在源数据库中存在。
    - 此时读者可能会问：会不会存在key丢失的情况？答案是：key丢失的情况是不可能发生的。如果MIGRATE命令在执行的过程中出现了其他错误，那么该命令可以保证这个key
  
2. 只存在于源数据库中。
 - 参数COPY：如果在MIGRATE命令中设置了COPY参数，则表示在转移之后不会删除源数据库中的key。
 - 参数REPLACE：如果在MIGRATE命令中设置了REPLACE参数，则表示在转移过程中会替换目标数据库中已经存在的key。
 - **返回值**：转移成功时返回OK；转移失败时返回相应的错误。
 - **操作实例**：
  先启动两个数据库实例（6379和6380），命令如下：
   -./redis - server --port 6379 &
   -./redis - server --port 6380 &
  然后开启两个客户端分别连接6379和6380，命令如下：
   -./redis - cli -p 6379
   -./redis - cli -p 6380
  在客户端6379中输入如下命令：
   - [root@localhost src]#./redis - cli -p 6379
   - 127.0.0.1:6379> SET article 'Sometimes all this is just hard to believe.'
     - OK
   - 127.0.0.1:6379> MIGRATE 127.0.0.1 6380 article 0 10 #将article键值对转移到127.0.0.1:6380数据库的0号数据库中，转移的最大时间间隔为10毫秒
     - OK
   - 127.0.0.1:6379> SET article1 'Thank you, my love, for coming into my life.'
     - OK
   - 127.0.0.1:6379> MIGRATE 127.0.0.1 6380 article1 2 10 COPY #将article1键值对转移到127.0.0.1:6380数据库的2号数据库中，转移的最大时间间隔为10毫秒，同时不删除源数据库中的键值对
     - OK
   - 127.0.0.1:6379> EXISTS article article1
     - (integer) 1 #表示article1键值对已存在
  然后到6380客户端查看，操作如下：
   - [root@localhost src]#./redis - cli -p 6380
   - 127.0.0.1:6380> GET article
     - (nil)
   - 127.0.0.1:6380> GET article
     - "Sometimes all this is just hard to believe."
   - 127.0.0.1:6380> GET article1
     - (nil)
   - 127.0.0.1:6380> SELECT 2
     - OK
   - 127.0.0.1:6380[2]> GET article1
     - "Thank you, my love, for coming into my life."

2. **MOVE命令：转移键值对到本地目标数据库**
 - **命令格式**：MOVE key db
 - MOVE命令用于将当前数据库中的key转移到指定的数据库db中，如果当前数据库中没有指定的key，那么MOVE命令什么也不做。如果当前数据库和给定数据库db中存在相同的key，那么MOVE命令没有任何效果。我们可以将MOVE命令的这一特性看作锁原语。
 - **返回值**：MOVE命令执行成功返回1，失败返回0。
 - **操作示例**：将article1键值对先后转移到3号、5号数据库中。
   - 127.0.0.1:6379> keys
     - 1) "article1"
   - 127.0.0.1:6379> GET article1
     - "Thank you, my love, for coming into my life."
   - 127.0.0.1:6379> MOVE article1 3 #将article1键值对转移到3号数据库中
     - (integer) 1
   - 127.0.0.1:6379> GET article1
     - (nil)
   - 127.0.0.1:6379> SELECT 3 #切换到3号数据库
     - OK
   - 127.0.0.1:6379[3]> GET article1
     - "Thank you, my love, for coming into my life."
   - 127.0.0.1:6379[3]> MOVE article1 5 #将3号数据库中的article1键值对转移到5号数据库中
     - (integer) 1
   - 127.0.0.1:6379[3]> GET article1
     - (nil)
   - 127.0.0.1:6379[3]> SELECT 5 #切换到5号数据库
     - OK
   - 127.0.0.1:6379[5]> GET article1
     - "Thank you, my love, for coming into my life."

3. **SORT命令：对键值对进行排序**
 - **命令格式**：SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern...]] [ASC | DESC] [ALPHA] [STORE destination]
 - SORT命令主要用于排序，它返回或保存给定列表、集合、有序集合key中经过排序的元素。排序默认以数字作为对象，值会被解释为double类型的浮点数，然后进行比较。SORT命令的参数众多，后面将会详细讲解，在这里先略过。
 - **返回值**：如果没有使用STORE参数，那么排序结果将会以列表的形式返回；如果使用了STORE参数，则将会返回排序结果的元素数量。
 - **操作示例**：将学生的分数（score）添加到列表score - list中，然后对分数（score）进行排序。
   - 127.0.0.1:6379[5]> LPUSH score - list 98 87 94 63 77
     - (integer) 5
   - 127.0.0.1:6379[5]> SORT score - list #升序排序
     - 1) "63"
     - 2) "77"
     - 3) "87"
     - 4) "94"
     - 5) "98"
   - 127.0.0.1:6379[5]> SORT score - list DESC #降序排序
     - 1) "98"
     - 2) "94"
     - 3) "87"
     - 4) "77"
     - 5) "63"

4. **TYPE命令：获取键对应值的类型**
 - **命令格式**：TYPE key
 - TYPE命令用于返回key所对应值的类型。
 - **返回值**：
   - 如果key不存在，则返回none。
   - 如果key所对应的值是字符串类型的，则返回string。
   - 如果key所对应的值是列表类型的，则返回list。
   - 如果key所对应的值是集合类型的，则返回set。
   - 如果key所对应的值是有序集合类型的，则返回zset。
   - 如果key所对应的值是哈希类型的，则返回hash。
 - **操作示例**：获取5号数据库中键对应值的类型。
   - 127.0.0.1:6379[5]> SADD age - set 24 23 22 25 23 #将学生的年龄添加到集合age - set中
     - (integer) 4
   - 127.0.0.1:6379[5]> HMSET student name '武恬' password '549871' age '22' #添加学生信息到哈希student中
     - OK
   - 127.0.0.1:6379[5]> KEYS * #获取5号数据库中所有的键
     - 1) "age - set"
     - 2) "student"
     - 3) "read"
     - 4) "red"
     - 5) "article1"
     - 6) "age"
     - 7) "message"
     - 8) "name"
     - 9) "really"
     - 10) "mobile"
     - 11) "reduce"
     - 12) "className"
     - 13) "class - name - 1"
     - 14) "girl - name"
     - 15) "userName"
     - 16) "redis"
     - 17) "score - list"
   - 127.0.0.1:6379[5]> TYPE mobile
     - string #键mobile是字符串类型的
   - 127.0.0.1:6379[5]> TYPE age - set
     - set #键age - set是集合类型的
   - 127.0.0.1:6379[5]> TYPE score - list
     - list #键score - list是列表类型的
   - 127.0.0.1:6379[5]> TYPE student
     - hash #键student是哈希类型的 
