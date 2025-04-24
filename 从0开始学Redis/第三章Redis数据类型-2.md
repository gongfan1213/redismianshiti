age
sex
height
weight
birthday
className
motto
127.0.0.1:6379[1]> HLEN student1
9
2. **HSTRLEN命令：统计域的值的字符串长度**
    - **命令格式**：HSTRLEN key field
    - HSTRLEN命令用于统计哈希表key中与给定域（field）相关联的值的字符串长度。当key或field不存在时，该命令返回0。
    - **返回值**：执行该命令后，将会返回一个整数，这个整数大于或等于0。
    - **统计哈希表student1中域的值的字符串长度，操作如下**：
```bash
127.0.0.1:6379[1]> HSTRLEN student1 stuName  #统计学生姓名的长度
9
127.0.0.1:6379[1]> HSTRLEN student1 stuID  #统计学生学号的长度
8
127.0.0.1:6379[1]> HSTRLEN student className  #统计一个不存在的哈希表中域的值的长度
0
127.0.0.1:6379[1]> HSTRLEN student1 className
16
127.0.0.1:6379[1]> HSTRLEN student1 motto
39
```
### 3.2.4 为哈希表中的域加上增量值
1. **HINCRBY命令：为哈希表中的域加上增量值**
    - **命令格式**：HINCRBY key field increment
    - HINCRBY命令用于为哈希表key中field的值加上增量值（increment）。这个增量值可以是一个负数，这相当于对这个field的值进行减法操作。如果key不存在，则将会创建一个新的哈希表key，然后继续执行HINCRBY命令。而如果field不存在，则将会把field的值初始化为0，然后执行命令。在执行HINCRBY命令时，必须保证field是数值类型的。如果对一个存储字符串的field执行HINCRBY命令，则将会报错。HINCRBY命令操作的值被限制在64位（bit）有符号数字表示范围之内。
    - **返回值**：返回执行命令之后的新值，也就是哈希表key中域（field）的值。
    - **为哈希表student1中的年龄、身高、体重、班级加上增量值3，操作如下**：
```bash
127.0.0.1:6379[1]> HINCRBY student1 age 3
25
127.0.0.1:6379[1]> HINCRBY student1 height 3
174
127.0.0.1:6379[1]> HINCRBY student1 weight 3
78
127.0.0.1:6379[1]> HINCRBY student1 className 3  #为字符串类型的值加上增量值，将会报错
ERR hash value is not an integer
```
2. **HINCRBYFLOAT命令：为哈希表中的域加上浮点数增量值**
    - **命令格式**：HINCRBYFLOAT key field increment
    - HINCRBYFLOAT命令用于为哈希表key中field的值加上浮点数增量值（increment）。如果key不存在，则该命令会先创建一个新的哈希表key，再创建field，最后执行浮点数加法操作。而如果field不存在，则该命令会先将field的值初始化为0，再执行浮点数加法操作。
    - 在Redis中，数字和浮点数都以字符串的形式进行保存。
    - 当域（field）的值不是字符串类型时，执行该命令将会报错。
    - 当域（field）的当前值或给定的增量值（increment）不是双精度浮点数时，执行该命令将会报错。
    - **返回值**：返回执行该命令之后field的新值。
    - **为哈希表student1中的身高、体重、班级加上浮点数增量值，操作如下**：
```bash
127.0.0.1:6379[1]> HINCRBYFLOAT student2 height 3.56
178.56
127.0.0.1:6379[1]> HINCRBYFLOAT student2 weight 4.61
77.61
127.0.0.1:6379[1]> HINCRBYFLOAT student2 className 2.54  #为字符串类型的值加上浮点数增量值，将会报错
ERR hash value is not a float
```
### 3.2.5 删除哈希表中的域
1. **HDEL命令：删除哈希表中的多个域**
    - **命令格式**：HDEL key field [field ...]
    - HDEL命令用于删除哈希表key中的一个或多个指定域（field），它会忽略不存在的域。
    - **返回值**：执行该命令后，将会返回被删除的域的数量，其中不包括被忽略的域。
    - **删除哈希表student5中的身高、体重、生日、班级等信息，操作如下**：
```bash
127.0.0.1:6379[1]> HDEL student5 height weight
2
127.0.0.1:6379[1]> HDEL student5 birthday className
2
127.0.0.1:6379[1]> HDEL student5 birthday className
0
```
2. **HEXISTS命令：判断哈希表中的域是否存在**
    - **命令格式**：HEXISTS key field
    - HEXISTS命令用于判断哈希表key中的field是否存在。
    - **返回值**：如果这个field存在，则返回1；如果这个哈希表key不存在，或者field不存在，则返回0。
    - **判断哈希表student5中的姓名、年龄、身高、班级等信息是否存在，操作如下**：
```bash
127.0.0.1:6379[1]> HEXISTS student5 stuName
1
127.0.0.1:6379[1]> HEXISTS student5 age
1
127.0.0.1:6379[1]> HEXISTS student5 height
0
127.0.0.1:6379[1]> HEXISTS student5 className
0
```
### 3.3 Redis数据类型之列表（List）命令
Redis的列表（List）数据类型可以被看作简单的字符串列表。列表按照插入顺序排序。在操作Redis的列表时，可以将一个元素插入这个列表的头部或尾部。一个列表大约可以存储2^32 - 1个元素。
我们仍以学生表3.1为例，切换到2号数据库，将学生信息以列表的形式添加到Redis数据库中，来逐个学习与List类型相关的命令。
#### 3.3.1 向列表中插入值
1. **LPUSH命令：将多个值插入列表头部**
    - **命令格式**：LPUSH key value [value ...]
    - LPUSH命令用于将一个或多个value值插入列表key的头部。如果同时插入多个value值，那么各个value值将会按照从左到右的顺序依次插入表头。例如，对于空列表list，执行命令LPUSH list a b c，列表key的值将是c b a。可以把列表想象成一只箱子，往里面装书（顺序是a、b、c），拿书时从上往下拿，顺序就是c、b、a。学过数据结构的读者应该很清楚，这个列表就相当于一个栈。
    - 当列表key不存在时，将会创建一个空列表，然后执行LPUSH命令。
    - 如果key存在，但它不是列表类型的，则执行LPUSH命令将会报错。
    - **返回值**：执行LPUSH命令后，返回列表的长度。
    - **将学生1~5的学号、年龄、性别、身高分别插入列表student1~student5中。为了保证顺序性，我们将学生的信息逆序插入。操作如下**：
```bash
127.0.0.1:6379[2]> LPUSH student1 171 '男' 22 20180001
4
127.0.0.1:6379[2]> LPUSH student2 175 '女' 24 20181762
4
127.0.0.1:6379[2]> LPUSH student3 168 '男' 23 20180023
4
127.0.0.1:6379[2]> LPUSH student4 170 '女' 23 20181120
4
127.0.0.1:6379[2]> LPUSH student5 166 '女' 22 20180097
4
```
当列表key不存在时，将会创建一个空列表，然后执行LPUSH命令。
如果key存在，但它不是列表类型的，则执行LPUSH命令将会报错。
**返回值**：执行LPUSH命令后，返回列表的长度。
**将学生1~5的学号、年龄、性别、身高分别插入列表student1~student5中。为了保证顺序性，我们将学生的信息逆序插入。操作如下**：
```bash
127.0.0.1:6379[2]> LPUSH student1 171 '男' 22 20180001
4
127.0.0.1:6379[2]> LPUSH student2 175 '女' 24 20181762
4
127.0.0.1:6379[2]> LPUSH student3 168 '男' 23 20180023
4
127.0.0.1:6379[2]> LPUSH student4 170 '女' 23 20181120
4
127.0.0.1:6379[2]> LPUSH student5 166 '女' 22 20180097
4
```
2. **RPUSH命令：将多个值插入列表尾部**
    - **命令格式**：RPUSH key value [value ...]
    - RPUSH命令用于将一个或多个value值插入列表key的表尾。如果同时插入多个value值，那么各个value值将会按照从左到右的顺序依次插入表尾。比如，对于空列表list，执行RPUSH list a b c命令之后，列表key的值将是a b c。
    - RPUSH list a b c命令相当于RPUSH list a、RPUSH list b、RPUSH list c。
    - 如果key不存在，则将会创建一个空列表，然后执行RPUSH命令。
    - 如果key存在，但它不是列表类型的，则执行RPUSH命令将会返回一个错误。
    - **返回值**：执行RPUSH命令后，返回列表的长度。
    - **将学生1~5的体重、生日分别插入列表student1~student5的表尾，操作如下**：
```bash
127.0.0.1:6379[2]> RPUSH student1 75 1996-02-14
6
127.0.0.1:6379[2]> RPUSH student2 73 1994-04-23
6
127.0.0.1:6379[2]> RPUSH student3 67 1995-08-18
6
127.0.0.1:6379[2]> RPUSH student4 64 1995-03-15
6
127.0.0.1:6379[2]> RPUSH student5 63 1996-09-10
6
```
3. **LINSERT命令：插入一个值到列表中**
    - **命令格式**：LINSERT key BEFORE | AFTER pivot value
    - LINSERT命令用于向列表中插入一个值，也就是将值value插入列表key当中，这个值的位置在值pivot之前或之后。在列表key中，当pivot这个值不存在时，执行该命令无效。当列表key不存在时，key将被看作空列表，执行该命令无效。而当key不是列表类型时，将返回一个错误。
    - **返回值**：执行该命令，如果成功，则返回插入操作完成之后的列表长度。如果只有pivot不存在，则返回 - 1。而如果key不存在，或者是空列表，则返回0。
    - **将学生1~5的班级信息分别插入列表student1~student5的年龄的后面，操作如下**：
```bash
127.0.0.1:6379[2]> LINSERT student1 AFTER 22 '软件工程1班'
7
127.0.0.1:6379[2]> LINSERT student2 AFTER 24 '网络工程1班'
7
127.0.0.1:6379[2]> LINSERT student3 AFTER 23 '软件工程2班'
7
127.0.0.1:6379[2]> LINSERT student4 AFTER 23 '信息管理1班'
7
127.0.0.1:6379[2]> LINSERT student5 AFTER 22 '软件工程2班'
7
```
4. **LPUSHX命令：将值插入列表头部**
    - **命令格式**：LPUSHX key value
    - LPUSHX命令用于将value值插入列表key的头部，此时key必须存在，并且是列表类型的。LPUSHX命令与LPUSH命令相反，当key不存在时，LPUSHX命令不会创建一个新的空列表，它什么也不做。
    - **返回值**：LPUSHX命令执行成功之后，返回列表key的长度。
    - **将学生1~5的姓名分别插入列表student1~student5的表头，操作如下**：
```bash
127.0.0.1:6379[2]> LPUSHX student1 '刘河飞'
8
127.0.0.1:6379[2]> LPUSHX student2 '赵雨梦'
8
127.0.0.1:6379[2]> LPUSHX student3 '宋飞'
8
127.0.0.1:6379[2]> LPUSHX student4 '陈慧'
8
127.0.0.1:6379[2]> LPUSHX student5 '孙玉'
8
```
5. **RPUSHX命令：将值插入列表尾部**
    - **命令格式**：RPUSHX key value
    - RPUSHX命令用于当且仅当key存在并且是列表类型时，将value值插入列表key的表尾。RPUSHX命令与RPUSH命令恰好相反，当key不存在时，它什么也不做，也不会创建空列表。
    - **返回值**：执行RPUSHX命令后，返回列表key的长度。
    - **将学生1~5的个人爱好信息分别插入列表student1~student5的表尾，操作如下**：
```bash
127.0.0.1:6379[2]> RPUSHX student1 '游泳'
9
127.0.0.1:6379[2]> RPUSHX student2 '旅游'
9
127.0.0.1:6379[2]> RPUSHX student3 '绘画'
9
127.0.0.1:6379[2]> RPUSHX student4 '音乐'
9
127.0.0.1:6379[2]> RPUSHX student5 '电影'
9
```
6. **LSET命令：修改列表元素值**
    - **命令格式**：LSET key index value
    - LSET命令用于设置下标为index的列表key的值为value。当下标index参数超出范围时，将会返回错误；当列表key为空时，也会返回错误。
    - **返回值**：如果LSET操作成功，则返回OK；否则返回错误。
    - **修改学生1的年龄、身高信息，修改学生3的学号、班级信息，操作如下**：
```bash
127.0.0.1:6379[2]> LSET student1 2 25  #学生1的年龄下标为2
OK
127.0.0.1:6379[2]> LSET student1 5 175  #学生1的身高下标为5
OK
127.0.0.1:6379[2]> LSET student3 1 20181123  #学生3的学号下标为1
OK
127.0.0.1:6379[2]> LSET student3 3 '车辆工程1班'  #学生3的班级下标为3
OK
```
### 3.3.2 获取列表元素
1. **LLEN命令：统计列表的长度**
    - **命令格式**：LLEN key
    - LLEN命令用于统计列表key的长度。当key不存在时，key将被视为空列表，返回0。当key不是列表类型时，返回一个错误。
    - **返回值**：执行该命令后，将会返回列表key的长度。
    - **分别统计学生列表student1~student5的长度，操作如下**：
```bash
127.0.0.1:6379[2]> LLEN student1
9
127.0.0.1:6379[2]> LLEN student2
9
```
```bash
127.0.0.1:6379[2]> LLEN student3
9
127.0.0.1:6379[2]> LLEN student4
9
127.0.0.1:6379[2]> LLEN student5
9
```
学生1 - 5的信息都是由姓名、学号、年龄、性别、身高、体重、生日、班级、爱好组成的，因此列表的长度都是9。
2. **LINDEX命令：获取列表元素的值**
    - **命令格式**：LINDEX key index
    - LINDEX命令用于获取列表key中下标为index的元素。index参数以0表示列表中的第一个元素，以1表示列表中的第二个元素，以此类推。index参数可以为负数，为 - 1时表示列表中的最后一个元素，为 - 2时表示列表中的倒数第二个元素，以此类推。当key不是列表类型时，返回一个错误。
    - **返回值**：当列表存在时，执行该命令后，返回列表中下标为index的元素。当index参数不在列表的范围之内（大于或小于列表范围）时，执行该命令后，将会返回nil。
    - **获取列表student1中下标为2和5的值，获取列表student3中下标为1和3的值，操作如下**：
```bash
127.0.0.1:6379[2]> LINDEX student1 2
25
127.0.0.1:6379[2]> LINDEX student1 5
175
127.0.0.1:6379[2]> LINDEX student3 1
20181123
127.0.0.1:6379[2]> LINDEX student3 3
车辆工程1班
```
获取到的值信息正是我们使用LSET命令修改后的值信息。
3. **LRANGE命令：获取列表指定区间内的元素**
    - **命令格式**：LRANGE key start end
    - LRANGE命令用于获取列表key指定区间内的元素，区间从start开始，到end结束。参数start和end都以0为底，即0表示列表中的第一个元素，1表示列表中的第二个元素，以此类推。参数start和end也可以是负数，即 - 1表示列表中的最后一个元素， - 2表示列表中的倒数第二个元素，以此类推。
    - 当参数start和end的值超出列表的下标值时，不会引起错误。
    - 当参数start的值大于列表的最大下标end值时，执行LRANGE命令会返回一个空列表。
    - 当设定的参数值比下标end值还要大时，Redis将会把这个设定的参数作为列表的end值（最大值）。
    - **返回值**：返回一个包含指定区间内的元素的列表。
    - **获取列表student1、student3指定区间内的元素，操作如下**：
```bash
127.0.0.1:6379[2]> LRANGE student1 0 -1  #获取列表student1中的所有元素
刘河飞
20180001
25
软件工程1班
男
175
75
1996-02-14
游泳
127.0.0.1:6379[2]> LRANGE student1 3 9  #获取列表student1中下标为3~9的元素
软件工程1班
男
175
75
1996-02-14
游泳
127.0.0.1:6379[2]> LRANGE student1 0 5  #获取列表student1中下标为0~5的元素
刘河飞
20180001
25
软件工程1班
男
175
127.0.0.1:6379[2]> LRANGE student3 10 20  #获取超出列表student3之外的元素，返回空
127.0.0.1:6379[2]> LRANGE student3 5 -1  #获取列表student3中下标为5~ - 1的元素
168
67
1995-08-18
绘画
```
### 3.3.3 删除列表元素
1. **LPOP命令：返回并删除列表的头元素**
    - **命令格式**：LPOP key
    - LPOP命令用于返回列表key的头元素，同时把这个头元素删除。
    - **返回值**：执行该命令后，将会返回列表的头元素。如果key不存在，则将会返回nil。
    - **删除学生列表student5的头元素，操作如下**：
```bash
127.0.0.1:6379[2]> LRANGE student5 0 -1  #获取列表student5中的所有元素
孙玉
20180097
22
软件工程2班
女
166
63
1996-09-10
电影
127.0.0.1:6379[2]> LPOP student5  #删除列表student5的头元素
孙玉
127.0.0.1:6379[2]> LPOP student5  #删除列表student5的头元素
20180097
127.0.0.1:6379[2]> LRANGE student5 0 -1  #查看删除头元素后的剩余元素
22
软件工程2班
女
166
63
1996-09-10
电影
```
2. **RPOP命令：返回并删除列表的尾元素**
    - **命令格式**：RPOP key
    - RPOP命令用于返回列表key的尾元素，并把这个尾元素删除。
    - **返回值**：当列表key存在时，执行RPOP命令将会返回表尾的元素。当列表key不存在时，将会返回nil。
    - **删除学生列表student5的尾元素，操作如下**：
```bash
127.0.0.1:6379[2]> LRANGE student5 0 -1  #获取列表student5中的所有元素
22
软件工程2班
女
166
63
1996-09-10
电影
127.0.0.1:6379[2]> RPOP student5  #删除列表student5的尾元素
电影
127.0.0.1:6379[2]> RPOP student5  #删除列表student5的尾元素
1996-09-10
127.0.0.1:6379[2]> LRANGE student5 0 -1  #查看删除尾元素后的剩余元素
22
软件工程2班
女
166
63
```
3. **BLPOP命令：在指定时间内删除列表的头元素**
    - **命令格式**： （此处文档未完整显示，无法准确提取）


**BLPOP key [key ...] timeout**
BLPOP命令是列表的阻塞式弹出原语，它是命令LPOP的阻塞版本。当列表中没有任何元素key被弹出时，连接将被BLPOP阻塞，直到等待超时或有可弹出元素为止。
设定多个参数key时，将会按照参数key出现的先后顺序来依次检查各个列表，弹出第一个非空列表的头元素。
**返回值**：如果给定的列表为空，则返回nil。如果给定的列表不为空，则将会返回一个包含两个元素的列表，列表中的第一个元素是要被弹出元素所对应的key，第二个元素是被弹出元素的值。
BLPOP命令存在两种行为：阻塞行为和非阻塞行为。
- **阻塞行为**：当所有给定的key都不存在或包含空列表时，该命令将会阻塞连接，直到等待超时，或者在另一个客户端中对给定的key执行RPUSH或LPUSH命令为止。其中，参数timeout是一个以秒为单位的超时时间值，当timeout取值为0时，这个阻塞时间可以无限延长。
- **在指定时间内，删除列表student5的头元素，操作如下**：
```bash
127.0.0.1:6379[2]> LRANGE student5 0 -1  #获取列表student5中的所有元素
22
软件工程2班
女
166
63
1996-09-10
127.0.0.1:6379[2]> BLPOP student5 5  #在5秒内，删除列表student5的头元素
22
127.0.0.1:6379[2]> EXISTS student6  #判断列表student6是否存在
0
127.0.0.1:6379[2]> BLPOP student6 300  #在300秒内，删除列表student6的头元素。开启另一个客户端，使用LPUSH命令添加元素到列表student6中；否则BLPOP命令会被阻塞
student6
一个好人
127.0.0.1:6379[2]> BLPOP student6 200
student6
20180103
```
4. **BRPOP命令：在指定时间内删除列表的尾元素**
    - **命令格式**：BRPOP key [key ...] timeout
    - BRPOP命令是列表key的阻塞式命令。当给定列表内没有任何元素可以返回时，连接将被BRPOP命令阻塞，直到等待超时或发现可返回的元素为止。BRPOP命令是RPOP命令的阻塞版本。当同时给定多个参数key时，将会按照参数key的先后顺序检查各个列表，返回第一个非空列表的尾元素。timeout参数用于设定时长。
    - **返回值**：如果在指定的timeout时间内没有返回任何元素，则将会返回nil和等待时长。而如果在timeout时间内返回一个列表，那么这个列表中的第一个元素表示被返回元素所属的key，第二个元素表示被返回元素的值。
    - **在指定时间内，删除列表student5的尾元素，操作如下**：
```bash
127.0.0.1:6379[2]> LRANGE student5 0 -1
软件工程2班
女
166
63
1996-09-10
127.0.0.1:6379[2]> BRPOP student5 20  #在20秒内，删除列表student5的尾元素
student5
63
127.0.0.1:6379[2]> BRPOP student5 200
student5
166
```
5. **LREM命令：删除指定个数的元素**
    - **命令格式**：LREM key count value
    - LREM命令用于根据参数count的值，删除列表key中与指定参数value相等的元素。
      - 当count等于0时，表示删除列表key中所有与value相等的元素。
      - 当count大于0时，表示从列表key的表头开始向表尾搜索，删除与value相等的元素，删除的数量为count个。
      - 当count小于0时，表示从列表key的表尾开始向表头搜索，删除与value相等的元素，删除的数量为count的绝对值个。
    - **返回值**：当列表key存在时，执行该命令后，返回被删除的元素数量。当列表key不存在时，就是一个空列表，该命令始终返回0。
    - **分别删除列表student5的1个、2个元素，操作如下**：
```bash
127.0.0.1:6379[2]> LRANGE student5 0 -1
软件工程2班
女
127.0.0.1:6379[2]> LREM student5 1 '女'
1
127.0.0.1:6379[2]> LREM student5 2 '女'
0
```
6. **LTRIM命令：在指定区间内修剪列表**
    - **命令格式**：LTRIM key start stop
    - LTRIM命令用于对一个列表进行修剪（trim），比如，去除不必要的空格，让列表key只保留指定区间内的元素，不在这个区间内的元素将会被删除 。 
