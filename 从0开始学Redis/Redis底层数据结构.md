### 第7章 Redis底层数据结构
本章将深入Redis的底层实现，讲解Redis中字符串、链表、字典、对象的底层实现原理，剖析它们底层实现的数据结构，帮助读者熟悉它们的实现过程、相关的API，以及对象的编码方式等。

#### 7.1 Redis简单动态字符串
我们已经知道，Redis数据库是由C语言编写实现的，它底层实现的代码具有C语言的特点及语法。C语言中具有字符串数据类型，Redis数据库中也有。但是Redis数据库并没有直接使用C语言中的字符串表示，而是自己重新构建了一种名为简单动态字符串（SDS）的抽象类型，并将其用作Redis的默认字符串表示。

##### 7.1.1 SDS的实现原理
在Redis中，C语言字符串通常用作字符串字面量，用在对字符串值不需要修改的地方，比如打印日志。当Redis需要一个可以被修改的字符串值时，它就会使用SDS来表示字符串值。Redis数据库采用SDS实现底层字符串值的键值对。执行以下命令：
```
127.0.0.1:6379>SET userName "liuhefei"
OK
```
在将字符串userName的值添加到Redis数据库的过程中，会创建一个新的键值对，这个键值对的键是一个字符串对象，其底层实现是一个保存着字符串“userName”的SDS；而这个键值对的值也是一个字符串对象，其底层实现是一个保存着字符串“liuhefei”的SDS。
SDS不仅可以用来保存数据库中的字符串值，还可以用于实现AOF模块下的AOF缓冲区（Buffer），以及实现客户端状态的输入缓冲区。
SDS是一个C语言结构体，位于Redis安装包的src目录下，每个sds.h/sdshdr结构表示一个SDS值，它的底层源代码如下：
```c
typedef char *sds;
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
通过SDS的底层源代码可以看出，它定义了多个不同类型的结构体，来适应不同场景的需求。它的结构体有4个参数，分别如下：
- len：len属性记录了buf数组中已使用的字节数量，也就是SDS所保存字符串的长度。比如，len属性值为7，表示这个SDS中保存了一个7字节长的字符串。
- alloc：alloc属性记录了buf数组中没有使用的字节数量。比如，alloc属性值为0，表示没有为这个SDS分配任何使用空间。 
- flags：flags属性是一个标识。 
- buf[]：buf属性是一个char类型的数组，用于以二进制的形式保存字符串。比如，保存的字符串是“liuhefei”，buf数组的保存形式是'l'、'i'、'u'、'h'、'e'、'f'、'e'、'i' 8个字符，而最后1字节用于保存字符 '\0'。

SDS采用了C语言中以空字符结尾的形式来保存字符串，保存空字符的1字节空间不计算到SDS的len属性中。在Redis中，SDS函数会为这个空字符另外分配1字节空间，并且会将这个空字符添加到字符串的结尾。Redis采用C语言字符串结尾添加空字符的优点是，SDS函数可以直接使用一部分C语言字符串库函数中的函数。

C语言字符串中的字符必须符合某种编码（如ASCII）方式，并且字符串中除末尾可以有空字符之外，其他地方不能出现空字符。如果出现空字符，那么这个空字符将会被认为是一个字符串的结束标识。这些限制导致C语言字符串只能保存文本数据，而不能保存二进制文件，如音频、视频、图片及压缩文件等。因此，我们可以知道，C语言是二进制不安全的。而Redis作为一个键值对存储数据库，为了适应不同的存储需求，它不能原样地采用C语言的字符串语法格式，而是自己重新构建了简单动态字符串（SDS）的抽象类型，所以SDS对应的API是二进制安全的。所有SDS API在处理数据时，都会以二进制的方式来处理SDS存储到buf数组中的数据，这些数据在保存的过程中不会受到任何限制、过滤，数据写入时是什么样的，读取出来就是什么样的。

C语言字符串与SDS的区别总结如下：
（1）C语言字符串的API是二进制不安全的，可能会存在缓冲区溢出；它只能保存文本数据；可以使用<string.h>库中的所有函数；每修改一次字符串长度就要重新分配一下内存，修改N次就需要重新分配N次内存；获取一个字符串长度的复杂度为O(N)。
（2）SDS的API是二进制安全的，它不会造成缓冲区溢出；它可以保存文本数据或二进制数据；可以使用<string.h>库中的一部分函数；修改字符串长度N次最多需要重新分配N次内存；获取一个字符串长度的复杂度为O(1)。

##### 7.1.2 SDS API函数
SDS API函数列举如下。
- sdsnew函数：使用该函数创建一个SDS，这个SDS中包含给定的C语言字符串。
- sdsempty函数：使用该函数创建一个空的SDS，它里面没有任何东西。
- sdslen函数：该函数用于获取SDS中已经使用完的空间字节数。它可以通过SDS的len属性获取。
- sdsfree函数：该函数用于释放指定的SDS。
- sdsclear函数：该函数用于清空（删除）SDS中保存的字符串。
- sdsdup函数：该函数用于创建一个指定的SDS的副本，也就是复制一份。
- sdscat函数：该函数用于在SDS字符串的结尾拼接一个给定的C语言字符串。
- sdscatsds函数：该函数用于将给定的SDS字符串拼接到另一个SDS字符串的结尾。
- sdsavail函数：该函数用于获取SDS字符串空间未使用的字节数。这个值可以通过读取SDS的alloc属性来获取。
- sdscmp函数：该函数用于比较两个SDS字符串是否相同。
- sdstrim函数：该函数具有两个参数，一个参数是SDS字符串，另一个参数是C语言字符串。sdstrim函数会从SDS字符串中移除所有在C语言字符串中出现过的字符。 
- sdscpy函数：该函数用于将指定的C语言字符串复制到SDS里面，它会覆盖原有的字符串。
- sdsrange函数：该函数用于保留给定区间内的SDS数据，不在这个区间内的数据将会被删除或覆盖。
- sdsgrowzero函数：该函数用于将SDS字符串扩展到指定的长度，采用空字符填充。

#### 7.2 Redis链表
链表是一种最常用的数据结构，它由多个离散分配的节点组成，节点之间通过指针相连，每个节点都有一个前驱节点和后继节点，但第一个头节点没有前驱节点，最后一个尾节点没有后继节点。很多计算机高级语言都内置了链表结构，但是C语言中没有内置链表结构，因此Redis自己构建了链表结构，用于适应不同的业务类型。

##### 7.2.1 链表的实现原理
在Redis中，多处用到了链表结构，比如，列表键的底层实现，就是当一个列表键包含了许多元素，或者列表键包含的元素都是一些比较长的字符串时，Redis就会使用链表结构来作为列表键的底层实现。又如，Redis消息订阅发布、监视器、慢查询等相关功能的底层实现都采用了链表结构。除此之外，Redis服务器也使用链表来保存多个客户端的状态信息，以及采用链表来构建客户端的输出缓冲区等。

下面我们将多个学生的姓名添加到列表键students中，列表键students所包含的内容底层就是采用链表结构实现的，操作如下：
```
127.0.0.1:6379> DEL students
(integer) 1
127.0.0.1:6379> LPUSH students "liuyi" "xiaoer" "zhangsan" #添加多个学生
(integer) 3
127.0.0.1:6379> LPUSH students "lisi" "wangwu" "zhaoliu"
(integer) 6
127.0.0.1:6379> LPUSH students "tianqi" "huba" "lijiu"
(integer) 9
127.0.0.1:6379> LLEN students 
(integer) 9 #获取学生的人数
127.0.0.1:6379> LRANGE students 0 -1 #遍历所有的学生
1) "lijiu"
2) "huba"
3) "tianqi"
4) "zhaoliu"
5) "wangwu"
6) "lisi"
7) "zhangsan"
8) "xiaoer"
9) "liuyi"
```
1. **链表的实现**

每个链表节点都使用一个adlist.h/listNode结构来表示，adlist.h源码位于Redis安装目录的src文件夹下，部分源码为：
```c
typedef struct listNode {
    struct listNode *prev; //表示头节点，或者上一个节点
    struct listNode *next; //表示下一个节点
    void *value; //节点的值
} listNode;
```
![image](https://github.com/user-attachments/assets/8dd466d3-983d-4a40-b67e-da0e208d9e01)


多个listNode节点通过prev和next指针相连接，组成单向链表list，它们也可以组成双向链表，Redis采用的就是双向链表，如图7.1所示。
```c
typedef struct list {
    listNode *head; //表示链表头节点
    listNode *tail; //表示链表尾节点
    void *(*dup)(void *ptr); //链表节点值复制函数
    void (*free)(void *ptr); //链表节点值释放函数
    int (*match)(void *ptr, void *key); //对比函数，用于对比链表的节点值
    unsigned long len; //表示链表所含有的节点数量，也就是链表的长度
} list;
```
多个节点组成一个链表list，这个链表中含有表头指针head、表尾指针tail、链表长度len，以及dup函数、free函数、match函数。其中，
- dup函数用于复制链表节点所保存的值。 
- free函数用于释放链表节点所保存的值。 
- match函数根据输入的值来和链表中保存的节点值进行比较，看是否相等。

2. **链表的特点**

链表具有以下特点：

- 带有表头指针head、表尾指针tail及链表长度计数器len，这样获取链表的头节点、尾节点、长度就会比较方便。 
- Redis链表是双向链表，链表节点带有prev和next指针，可以很容易地获取到链表中的某个节点。


表头节点的prev指针和表尾节点的next指针都指向NULL。在访问链表的过程中，如果遇到NULL，则表示链表访问结束。
Redis链表可以保存各种不同类型的值，链表节点使用void*指针来保存节点的值。链表中的dup、free、match函数可以为节点值设置类型特定函数，实现节点值的复制、释放、比较操作。

#### 7.2.2 链表API函数
链表API函数列举如下。
- listCreate函数：该函数用于创建一个空的新链表，它不包含任何节点元素。
- listFirst函数：该函数用于获取链表的表头节点，也可以通过链表的head属性获得。
- listLast函数：该函数用于获取链表的表尾节点，也可以通过链表的tail属性获得。 
- listLength函数：该函数用于获取链表的长度，也可以通过链表的len属性获得。 
- listPrevNode函数：该函数用于获取给定节点的上一个节点，也可以通过节点的prev属性获得。 
- listNextNode函数：该函数用于获取给定节点的下一个节点，也可以通过节点的next属性获得。 
- listNodeValue函数：该函数用于获取给定节点中保存的值，也可以通过节点的value属性获得。 
- listAddNodeHead函数：该函数用于在指定链表的表头插入一个给定的节点。 
- listAddNodeTail函数：该函数用于在指定链表的表尾插入一个给定的节点。 
- listInsertNode函数：该函数用于将一个包含给定值的新节点插入指定节点的前面或后面。 
- listIndex函数：该函数用于返回链表在给定索引上的节点。 
- listSearchKey函数：该函数用于在链表中查找给定的节点，查找到就将它返回。 
- listRotate函数：该函数用于获取链表的表尾节点，然后将获取到的节点插入链表的表头，成为新的表头节点。 
- listDelNode函数：该函数用于删除链表中指定的节点。 
- listDup函数：该函数用于复制一个给定链表的副本。 
- listRelease函数：该函数用于释放指定的链表，包含这个链表的全部节点。 
- listSetDupMethod函数：该函数用于将指定的函数设置为链表的节点值复制函数。这个复制函数可以使用链表的dup属性获得。 
- listGetDupMethod函数：该函数用于获取链表中正在使用的节点值复制函数。 
- listSetFreeMethod函数：该函数用于将指定的函数设置为链表的节点值释放函数。这个释放函数可以使用链表的free属性获得。 
- listGetFree函数：该函数用于获取链表中正在使用的节点值释放函数。 
- listSetMatchMethod函数：该函数用于将指定的函数设置为链表的节点值对比函数。这个对比函数可以使用链表的match属性获得。 
- listGetMatchMethod函数：该函数用于获取链表中正在使用的节点值对比函数。

### 7.3 Redis压缩列表
Redis的压缩列表（ziplist）是列表键和哈希键的底层实现之一。

#### 7.3.1 压缩列表的实现原理
当一个列表键包含的元素比较少时，且这些列表元素要么是小整数值，要么是短字符串，Redis就会采用压缩列表来实现这个列表键的底层。

当一个哈希键包含的键值对比较少时，且每个键值对的键和值要么是小整数值，要么是短字符串，Redis就会采用压缩列表来实现这个哈希键的底层。

向Redis数据库中添加一条用户信息，用于创建一个底层采用压缩列表实现的哈希键，操作如下：
```
127.0.0.1:6379> HMSET user userName "liuhefei" passWord "123456" age 24 height 172 weight 140 #向数据库中添加一条用户信息，包括用户名、密码、年龄、身高、体重
OK
127.0.0.1:6379> HMGET user userName passWord age height weight #获取用户的用户名、密码、年龄、身高、体重
1) "liuhefei"
2) "123456"
3) "24"
4) "172"
5) "140"
127.0.0.1:6379> OBJECT ENCODING user #查看键user的底层实现
"ziplist" #压缩列表
```
1. **压缩列表模型图**

![image](https://github.com/user-attachments/assets/cbc2e3c5-f396-476b-b56a-8c1740876be4)


在Redis中，压缩列表是一个顺序型数据结构，它由一系列特殊编码的连续内存块组成，它是为节省内存而开发的。一个压缩列表可以包含任意多个节点，每个节点都可以保存一个整数值或者一个字节数组。压缩列表模型图如图7.2所示。
- zbytes属性：该属性记录了整个压缩列表所占用的内存字节数，是一个uint32_t类型的4字节数值。压缩列表在进行内存分配或计算zlend的位置时，才会使用zbytes。
- ztail属性：该属性记录了压缩列表的表尾节点距离压缩列表的起始地址的字节数，是一个uint32_t类型的4字节数值。通过ztail属性的值，程序不需要遍历压缩列表，就可以确定表尾节点的地址。
- zllen属性：该属性记录了整个压缩列表所包含的节点数量，它是一个uint16_t类型的2字节数值。当zllen属性的值小于UINT16_MAX(65535)时，这个属性的值就是压缩列表所包含的节点数量。当zllen属性的值等于UINT16_MAX时，需要遍历整个压缩列表才能计算出压缩列表所包含的节点数量。
- entry属性：该属性表示压缩列表的节点，一个压缩列表有任意多个节点，节点的数量取决于节点保存的内容。
- zlend属性：该属性是一个特殊值0xFF（对应的十进制数是255），用于标识压缩列表的末端，是一个uint8_t类型的1字节数值。

2. **压缩列表节点模型**

每个压缩列表节点都可以保存一个整数值或一个字节数组。

可以保存的整数值有多种，具体如下：
- 4位长、介于0~12之间的无符号整数。
- 1字节长的有符号整数。
- 3字节长的有符号整数。
- int16_t类型整数。
- int32_t类型整数。
- int64_t类型整数。
可以保存的字节数组可以是以下3种长度中的一种：
- 长度小于或等于63（2^6 - 1）字节的字节数组。
- 长度小于或等于16 383（2^14 - 1）字节的字节数组。
- 长度小于或等于4 294 967 295（2^32 - 1）字节的字节数组。
如图7.3所示为压缩列表节点模型图。

![image](https://github.com/user-attachments/assets/ef7f7f73-6dba-4169-a064-36fd82f0080d)


每个压缩列表节点都有previous_entry_length、encoding、content 3个属性。
- previous_entry_length属性：用于记录压缩列表中前一个节点的长度，这个长度可以是1字节，也可以是5字节。该属性以字节为单位。当压缩列表的前一个节点的长度小于254字节时，previous_entry_length属性的长度为1字节，这个字节保存了前一个节点的长度。当压缩列表的前一个节点的长度大于或等于254字节时，previous_entry_length属性的长度为5字节，其中第一个字节会被设置为0xFF（对应十进制数255），后面的4个字节用于保存前一个节点的长度。
- encoding属性：该属性记录了节点的content属性所保存数据的类型及长度。
  - 1字节、2字节或5字节长，这些字节数组编码的值的最高位是00、01或10。以这种编码方式表示节点的content属性保存的是字节数组，这个字节数组的长度由编码除去最高两位之后的其他位记录。
    - 以00开头的编码方式，1字节，表示content属性保存的值是长度小于或等于63字节的字节数组。
    - 以01开头的编码方式，2字节，表示content属性保存的值是长度小于或等于16 383字节的字节数组。
    - 以10开头的编码方式，5字节，表示content属性保存的值是长度小于或等于4 294 967 295字节的字节数组。
  - 1字节长，值的最高位以11开头的整数编码，表示content属性保存的数据是整数数值，整数数值的类型和长度由编码除去最高两位之后的其他位记录。编码方式不同，content属性保存的整数值的类型也就不同。
如表7.1所示为1字节整数编码对应的content属性中保存的值类型。

|编码|content属性中保存的值类型|
| ---- | ---- |
|1100 0000|int16_t类型的整数|
|1101 0000|int32_t类型的整数|
|1110 0000|int64_t类型的整数|
|1111 0000|24位有符号整数|
|1111 1110|8位有符号整数|
|1111 xxxx|表示这个压缩列表的节点没有content属性，编码本身的xxxx 4位已经保存了一个介于0~12之间的值|
- content属性：该属性用于保存压缩列表节点的值。节点的值可以是一个整数，也可以是一个字节数组，而节点的encoding属性决定了这个值的类型和长度。

#### 7.3.2 压缩列表API函数
压缩列表API函数列举如下。
- ziplistNew函数：该函数用于创建一个新的压缩列表。
- ziplistInsert函数：该函数用于将包含给定值的新节点插入压缩列表指定节点的后面。
- ziplistPush函数：该函数用于创建一个包含给定值的新节点，并将这个新节点插入压缩列表的表头或表尾。
- ziplistNext函数：该函数用于获取压缩列表指定节点的下一个节点。
- ziplistPrev函数：该函数用于获取压缩列表指定节点的上一个节点。
- ziplistIndex函数：该函数用于获取压缩列表在给定索引上的节点。
- ziplistFind函数：该函数用于在压缩列表中查找，并返回包含了指定值的节点。
- ziplistGet函数：该函数用于获取给定节点所保存的值。
- ziplistDelete函数：该函数用于在压缩列表中删除指定的值。
- ziplistDeleteRange函数：该函数用于删除压缩列表在给定索引上的连续多个节点。
- ziplistLen函数：该函数用于获取压缩列表所包含的节点数量。
- ziplistBlobLen函数：该函数用于获取压缩列表目前所占用的内存字节数。

### 7.4 Redis快速列表
在Redis 3.2版本中引入了新的数据结构——快速列表（quicklist），用于列表的底层实现。

#### 7.4.1 快速列表的实现原理
将多个学生的数学成绩添加到数据库的列表math-score中，并使用命令查看列表的底层实现，操作如下：
```
127.0.0.1:6379> RPUSH math-score 79 100 99 76 88 67 84 91 78 88 #添加学生成绩到列表math-score中
(integer) 10
127.0.0.1:6379> LRANGE math-score 0 -1 #遍历学生成绩
1) "79"
2) "100"
3) "99"
4) "76"
5) "88"
6) "67"
7) "84"
8) "91"
9) "78"
10) "88"
127.0.0.1:6379> OBJECT ENCODING math-score #查看列表键math-score的底层实现
"quicklist" #快速列表
```
快速列表是由压缩列表组成的双向链表，链表的每个节点都以压缩列表的结构来保存数据。压缩列表有任意多个entry节点，用于保存数据，因此快速列表可以保存更多的数据，你可以理解为它保存的是一片数据。

快速列表的定义位于Redis安装目录下src文件夹中的quicklist.h文件中，定义如下：
```c
typedef struct quicklist {
    //指向头部(最左边)quicklist节点的指针
    quicklistNode *head;
    //指向尾部(最右边)quicklist节点的指针
    quicklistNode *tail;
    //ziplist中的entry节点计数器
    unsigned long count; /* total count of all entries in all ziplists */
    //quicklist的quicklistNode节点计数器
    unsigned int len; /* number of quicklistNodes */
    //保存ziplist的大小，配置文件设定，占16bits
    int fill : 16; /* fill factor for individual nodes */
    //保存压缩程度值，配置文件设定，占16bits，0表示不压缩
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off */
} quicklist;
```
在快速列表的定义结构中，有fill和compress两个属性，其中“:”是位域运算符，表示fill占int类型32位中的16位，而compress也占int类型32位中的16位。
fill和compress属性在Redis的配置文件redis.conf中进行设置。
- fill属性对应的配置参数是list-max-ziplist-size-2。
list-max-ziplist-size属性具有多个值，具体含义如下。
  - 当设置为-1时，表示每个quicklistNode节点的ziplist字节大小不能超过4KB（建议使用）。
  - 当设置为-2时，表示每个quicklistNode节点的ziplist字节大小不能超过8KB（默认配置）。 
  - 当设置为-3时，表示每个quicklistNode节点的ziplist字节大小不能超过16KB（一般不建议使用）。 
  - 当设置为-4时，表示每个quicklistNode节点的ziplist字节大小不能超过32KB（不建议使用）。 
  - 当设置为-5时，表示每个quicklistNode节点的ziplist字节大小不能超过64KB（正常工作量不建议使用）。 
  - 当设置为正数时，表示ziplist结构最多包含的entry节点个数，最大值为2^15。
- compress属性对应的配置参数是list-compress-depth 0。
list-compress-depth属性具有多个值，具体含义如下。
  - 当设置为0时，表示列表不压缩（默认设置）。 
  - 当设置为1时，表示快速列表除两端各有1个节点不压缩之外，其他的节点都压缩。 
  - 当设置为2时，表示快速列表除两端各有2个节点不压缩之外，其他的节点都压缩。 
  - 当设置为3时，表示快速列表除两端各有3个节点不压缩之外，其他的节点都压缩。
 

  快速列表节点的结构定义如下：
```c
typedef struct quicklistNode {
    struct quicklistNode *prev; //前驱节点指针
    struct quicklistNode *next; //后继节点指针
    //当不设置压缩数据参数recompress时指向ziplist结构
    //当设置压缩数据参数recompress时指向quicklistLZF结构
    unsigned char *zl; 
    //压缩列表ziplist的总长度
    unsigned int sz; /* ziplist size in bytes */
    //ziplist中包含的节点数，占16bits长度
    unsigned int count : 16; /* count of items in ziplist */
    //表示是否采用了LZF压缩算法压缩quicklist节点，1表示压缩了，2表示没压缩，占2bits长度
    unsigned int encoding : 2; /* RAW==1 or LZF==2 */
    //表示一个quicklistNode节点是否采用ziplist结构保存数据，2表示压缩了，1表示没压缩，默认是2，占2bits长度
    unsigned int container : 2; /* NONE==1 or ZIPLIST==2 */
    //标记quicklist节点的ziplist之前是否被解压缩过，占1bit长度
    //如果recompress为1，则等待被再次压缩
    unsigned int recompress : 1; /* was this node previous compressed? */
    //测试时使用
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    //额外扩展位，占10bits长度
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

#### 7.4.2 快速列表API函数
快速列表API函数列举如下。
- quicklistCreate函数：该函数用于创建一个新的快速列表。
- quicklistSetCompressDepth函数：该函数用于对指定的快速列表设置压缩程度。
- quicklistSetFill函数：该函数用于对指定的快速列表设置ziplist结构的大小。
- quicklistSetOptions函数：该函数用于为给定的快速列表设置压缩列表表头的fill和compress属性。 
- quicklistNew函数：该函数用于创建一个新的快速列表，并为其设置默认的参数。 
- quicklistCreateNode函数：该函数用于创建一个快速列表的节点（quicklistNode），并初始化。 
- quicklistCount函数：该函数用于统计ziplist结构中entry节点的个数。 
- quicklistRelease函数：该函数用于释放给定的快速列表。 
- quicklistPushHead函数：该函数用于追加一个entry节点到快速列表的头部。 
- quicklistPushTail函数：该函数用于追加一个entry节点到快速列表的尾部。如果追加失败，则新创建一个quicklistNode节点。 
- quicklistAppendZiplist函数：该函数用于为给定的快速列表追加一个quicklist节点。 
- quicklistDelEntry函数：该函数用于删除ziplist结构中的entry节点。 
- quicklistReplaceAtIndex函数：该函数用于在给定的快速列表中，将下标为index的值替换为data值。 
- quicklistDelRange函数：该函数用于在给定的快速列表中删除某个范围内的entry节点，返回1表示全部被删除，返回0表示删除失败。 
- quicklistRotate函数：该函数用于将尾quicklistNode节点的尾entry节点旋转到头quicklistNode节点的头部。 
关于快速列表的更多相关API，请读者自行查阅，在这里不再一一列举了。

### 7.5 Redis字典
Redis字典是一种用于保存Redis键值对的抽象数据结构，它是一种键值对的映射，有时也被称为符号表、关联数组等。在字典中，一个键与一个值进行关联，键与值是一一对应的，你可以理解为键映射为值，键与值进行关联，就称为键值对。字典中的每个键都是唯一的，不可能存在两个相同的键，我们可以根据这个键来查找它对应的值，也可以通过这个键来修改、删除它对应的值。

#### 7.5.1 字典的实现原理
字典作为一种常用的数据结构，内置在多种高级计算机语言中，但是C语言并没有内置字典数据结构，因此Redis根据需要构建了自己的字典。字典在Redis中得到了广泛应用，其中Redis数据库的底层就是采用字典实现的，对数据库中数据的增、删、改、查操作就是建立在字典的基础上的。同时，Redis哈希键的底层也是采用字典实现的。当一个哈希键包含的键值对比较多，或者键值对中的元素是比较长的字符串时，Redis就会采用字典作为哈希键的底层实现。比如，执行以下命令：
```
127.0.0.1:6379>SET name "liuhefei"
OK
```
键“name”和值“liuhefei”在数据库中就是以键值对的形式存储在字典中的。
下面向Redis数据库中再添加一条用户的详细信息，这条信息具体包含用户的用户名、密码、年龄、生日、身高、体重、电话号码、地址等多个键值对信息，操作如下：
```
127.0.0.1:6379> HMSET user1 userName "zhangsan" passWord "123456" age 20 birthday "1994-01-01" #添加用户信息到哈希表user1中
OK
127.0.0.1:6379> HMSET user1 height 172 weight 140 mobile "18296666666" address "beijing"
OK
127.0.0.1:6379> HGETALL user1 #获取哈希表user1中的键值对信息
1) "userName" #键
2) "zhangsan" #值
3) "passWord"
4) "123456"
5) "age"
6) "20"
7) "birthday"
8) "1994-01-01"
9) "height"
10) "172"
11) "weight"
12) "140"
13) "mobile"
14) "18296666666"
15) "address"
16) "beijing"
127.0.0.1:6379> HLEN user1 #获取哈希表user1的长度
(integer) 8
```
当哈希表user1中的键值对数量足够多时，Redis就会使用字典来存储这些信息，这个字典中包含多个键值对，例如，键值对的键为“userName”，值为“zhangsan”。
Redis采用哈希表实现了字典的底层。一个哈希表中有多个哈希表节点，而每个哈希表节点中就保存了字典中的一个键值对。Redis字典所使用的哈希表由dict.h/dictht结构定义，dict.h文件位于Redis安装包的src目录下，部分源码如下：
```c
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;
```
结构元素说明如下。
- table属性：这是一个哈希表数组，数组中的每个元素都是一个指向dict.h/dictEntry结构的指针，每个dictEntry结构中保存一个键值对。 
- size属性：该属性用于记录table数组的长度，也就是哈希表的大小。 
- sizemask属性：这是哈希表大小掩码，用来计算索引值，它的值总是等于size-1。sizemask属性与哈希值共同决定一个键应该放到table数组的哪个索引上。 
- used属性：该属性用于记录哈希表上已经存在的节点（键值对）数量。 

使用dictEntry结构表示哈希表节点，一个键值对保存在一个dictEntry结构中。dictEntry结构的部分源码如下：
```c
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```
结构元素说明如下。
- key属性：该属性用于保存键值对中的键。 
- v属性：该属性用于保存键值对中的值。键值对中的值可以是一个指针（*val），也可以是一个无符号的64位整数（u64），也可以是一个64位的整数（s64），还可以是一个double类型的值（d）。 
- next属性：该属性是一个指针，用于指向另一个哈希表节点。这个指针可以将多个哈希值相同的键值对连接在一起，还可以解决键冲突问题。 

Redis中的字典由dict.h/dict结构表示，位于Redis安装包的src目录下，源码如下：
```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
结构元素说明如下。
- type属性：该属性用于指向dictType结构，它是一个指针。每个dictType结构中保存一组用于操作特定类型键值对的函数。Redis会根据用途不同的字典，设置不同的类型特定函数。 
- privdata属性：该属性用于保存需要传递给那些类型特定函数的可选参数。type和privdata属性是为创建多态字典而设置的，二者针对不同类型的键值对。 
- ht[2]属性：该属性是一个包含两个数组元素的数组，数组中的每个元素都是一个dictht哈希表。通常，字典只使用ht[0]哈希表，而只有在对ht[0]哈希表进行rehash（重新散列）时，才会用到ht[1]哈希表。 
- rehashidx属性：该属性用于记录rehash目前的进度。如果现在没有进行rehash，那么它的值为-1。 
- iterators属性：该属性表示当前运行的迭代器数。 

dictType结构的源码定义如下：
```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```
结构元素说明如下。
- uint64_t (*hashFunction)(const void *key)：该函数用于计算哈希值，返回值类型为无符号整型。 
- void *(*keyDup)(void *privdata, const void *key)：该函数用于复制键，返回值类型为void（空类型）。 
- void *(*valDup)(void *privdata, const void *obj)：该函数用于复制值，返回值类型为void。 
- int (*keyCompare)(void *privdata, const void *key1, const void *key2)：该函数用于比对键值对中的键，看是否相同，返回值类型为void。 
- void (*keyDestructor)(void *privdata, void *key)：该函数用于销毁键值对中的键，返回值类型为void。 
- void (*valDestructor)(void *privdata, void *obj)：该函数用于销毁键值对中的值，返回值类型为void。 

如果要将一个新的键值对添加到字典中，那么程序需要先根据键值对中的键计算出哈希值和索引值，再根据索引值将包含新键值对的哈希表节点放到哈希表数组的指定索引上。Redis计算哈希值和索引值的步骤如下。
（1）使用字典设置的哈希函数，计算出键值对中键的哈希值。
```c
hash = dict -> type ->hashFunction(key);
```
（2）利用哈希表的sizemask属性和哈希值，计算出索引值。
```c
index = hash & dict ->ht[x].sizemask;
```
其中，ht[x]可以是ht[0]，也可以是ht[1]。
这就是Redis的哈希算法。
当数据库或哈希键的底层采用字典实现时，Redis计算键的哈希值会使用MurmurHash2算法实现。关于Redis哈希键及哈希表的更多相关底层知识，请读者自行查阅相关资料。

#### 7.5.2 字典API函数
字典API函数列举如下。 （此处未列出具体函数，原文未详细展开） 


- dictCreate函数：该函数用于创建一个新的字典。
- dictAdd函数：该函数用于添加一个给定的键值对到字典中。
- dictDelete函数：该函数根据给定的键删除字典中与之对应的键值对。
- dictFetchValue函数：该函数用于获取字典中给定键所对应的值。
- dictGetRandomKey函数：该函数用于将给定的键值对添加到字典中。如果这个字典中已经存在给定的键，那么旧值将会被新值覆盖。
- dictReplace函数：该函数用于释放给定的字典，包含字典中的所有键值对。换句话说，就是清空字典中的所有键值对。
- dictRelease函数：该函数用于释放给定的字典，包含字典中的所有键值对。 

### 7.6 Redis整数集合
Redis集合键的底层实现有多种方式，其中一种是整数集合（intset）。当一个集合只包含整数值元素，同时这个集合中的元素数量不是太多时，Redis就会采用整数集合来实现这个集合键的底层。
下面将多个学生的成绩添加到集合score中，并查看集合score的底层实现，操作如下：
```
127.0.0.1:6379> SADD score 60 75 70 80 89 90 100 92 81 73 #添加学生成绩到集合score中
(integer) 10
127.0.0.1:6379> SMEMBERS score #获取集合score中的所有元素
1) "60"
2) "70"
3) "73"
4) "75"
5) "80"
6) "81"
7) "89"
8) "90"
9) "92"
10) "100"
127.0.0.1:6379> OBJECT ENCODING score #查看集合score的底层实现
"intset"
```
执行OBJECT ENCODING score命令后，返回intset，说明集合score的底层实现采用的是整数集合。

#### 7.6.1 整数集合的实现原理
Redis底层使用整数集合来保存整数值类型的集合键（set集合）。整数集合（intset）可以保存int16_t、int32_t、int64_t类型的整数值，且整数集合元素不可重复。
位于Redis安装包的src目录下的intset.h/intset结构表示一个整数集合，该结构的定义如下：
```c
//整数集合结构
typedef struct intset {
    //编码方式
    uint32_t encoding;
    //集合中所包含的元素数量
    uint32_t length;
    //保存集合元素的数组
    int8_t contents[];
} intset;
```
属性说明如下。
- encoding属性：该属性用于定义整数集合的编码方式，不同的编码方式决定了整数集合可以保存什么类型的集合元素。它具有多个属性值，具体有INTSET_ENC_INT16、INTSET_ENC_INT32、INTSET_ENC_INT64。 
- length属性：该属性记录了整数集合所包含的元素数量，也就是contents数组的长度。 
- contents属性：该属性是一个声明为int8_t类型的数组，但是它并不保存任何int8_t类型的值。contents数组所能保存的集合元素的类型取决于encoding属性的值。
  - 当encoding属性的值为INTSET_ENC_INT16时，contents数组的类型为int16_t，数组中的每个元素都是一个int16_t类型的整数值，此时contents数组的大小为sizeof(int16_t)×length。这个数组所能存放的整数值范围是：最小值为-32 768，最大值为32 767。
  - 当encoding属性的值为INTSET_ENC_INT32时，contents数组的类型为int32_t，数组中的每个元素都是一个int32_t类型的整数值，此时contents数组的大小为sizeof(int32_t)×length。这个数组所能存放的整数值范围是：最小值为-2 147 483 648，最大值为2 147 483 647。 
  - 当encoding属性的值为INTSET_ENC_INT64时，contents数组的类型为int64_t，数组中的每个元素都是一个int64_t类型的整数值，此时contents数组的大小是sizeof(int64_t)×length。这个数组所能存放的整数值范围是：最小值为-9 223 372 036 854 775 808，最大值为9 223 372 036 854 775 807。 

如图7.4所示为一个含有6个int32_t类型整数值的整数集合。 

![image](https://github.com/user-attachments/assets/9b6f35b5-7a6c-4da2-a0a3-27fba7eaa3f0)


在图7.4中，encoding属性的属性值为INTSET_ENC_INT32，表示这个整数集合的底层实现为int32_t类型的数组，而这个数组中保存的元素都是int32_t类型的。length属性的值为6，表示这个整数集合包含6个集合元素。contents数组按照从小到大的顺序依次存储整数集合的元素，contents数组的大小为sizeof(int32_t)×length = 32×6 = 192（位）。 

现有一个类型为int16_t的整数集合，如果要向这个集合中添加类型为int64_t的集合元素，那么Redis的底层是如何实现的呢？ 

这个过程涉及整数集合元素的类型转化问题。int16_t类型的整数元素要转化为int64_t类型的整数元素，Redis的底层是这样实现的： 

（1）根据新添加元素的类型（这里为int64_t），扩展这个整数集合底层数组的空间大小，同时为这个新元素分配空间。 

（2）将底层数组原有的所有元素都转化为与新元素相同的类型（这里是将int16_t类型的元素转化为int64_t类型的元素），并将类型转化后的元素按照从小到大的顺序依次放置到正确的位置上，以保证底层数组的有序性。 

（3）将新元素添加到底层数组中，这个整数集合类型就由最初的int16_t类型转化为int64_t类型了。 

以上这个过程就是将一个低类型的整数集合转化为高类型的整数集合的过程。 

整数集合的类型由高到低为：int64_t > int32_t > int16_t。 

注意：Redis整数集合的底层并不支持高类型的整数集合转化为低类型的整数集合。一旦整数集合由低类型转化为高类型之后，整数集合的编码就会一直保持为转化后的状态，就不可能再转化为低类型的整数集合了。 

#### 7.6.2 整数集合API函数

整数集合API函数列举如下。 

- intsetNew函数：该函数用于创建一个新的整数集合。 

- intsetAdd函数：该函数用于将指定的整数元素添加到整数集合中。 

- intsetGet函数：该函数用于获取底层数组在给定索引上的元素。 

- intsetLen函数：该函数用于获取整数集合所包含的元素数量。 

- intsetRemove函数：该函数用于删除整数集合中指定的元素。 

- intsetFind函数：该函数用于判断给定的元素是否存在于整数集合中。 

- intsetRandom函数：该函数用于从整数集合中随机返回一个元素。 

- intsetBlobLen函数：该函数用于返回整数集合所占用的内存字节数。

### 7.7 Redis跳表
Redis跳表是一种有序数据结构，它的每个节点中具有多个指向其他节点的指针，这些指针可以实现快速访问节点的目的，它不仅支持快速节点查找，还可以通过操作批量处理节点。

#### 7.7.1 跳表的实现原理
Redis采用跳表实现了有序集合的底层。如果一个有序集合包含的元素数量众多，或者有序集合元素的成员是比较长的字符串，Redis就会采用跳表作为这个有序集合的底层实现。
下面的有序集合citys记录了中国600座城市的名称，以各城市的GDP（亿元）作为值，列举部分城市如下：
```
127.0.0.1:6379> ZRANGE citys 0 5 WITHSCORES
1) "haerbin-GDP"
2) "8645"
3) "dalian-GDP"
4) "9867"
5) "nanjing-GDP"
6) "10034"
7) "tianjin-GDP"
8) "11203"
9) "wuhan-GDP"
10) "12654"
11) "shenzhen-GDP"
12) "14321"
```
有序集合citys的所有数据都保存在一个跳表中，每个跳表节点都保存一座城市的GDP信息。跳表数据结构主要用在Redis的有序集合和集群节点中。
Redis的跳表由redis.h/zskiplistNode和redis.h/zskiplist结构定义，其中zskiplistNode结构用于表示跳表的节点，zskiplist结构用于保存跳表节点的相关信息，比如，保存节点的数量，以及指向表头节点和表尾节点的指针等。
zskiplist结构具有如下属性。
- header属性：该属性用于指向跳表的表头节点。
- tail属性：该属性用于指向跳表的表尾节点。
- level属性：该属性用于记录在目前的跳表内，除表头节点所在层数之外，层数最大的节点层数。
- length属性：该属性用于记录跳表的长度，也就是跳表中的节点数量，不包含表头节点。

zskiplistNode结构的定义如下：
```c
typedef struct zskiplistNode {
    struct zskiplistLevel {
        struct zskiplistNode *forward; //前进指针
        unsigned int span; //跨度
    } level[];
    //后退指针
    struct zskiplistNode *backward;
    //分数
    double score;
    //成员对象
    robj *obj;
} zskiplistNode;
```
属性说明如下。
- level属性：该属性是一个数组，表示跳表中的层。数组中可以包含多个元素，每个元素都包含一个指向其他节点的指针，程序通过这些层可以快速地访问到其他节点，层数越多，访问其他节点的速度越快。在创建新的跳表节点的时候，程序会根据幂次定律（越大的数，出现的概率越小）随机生成一个介于1～32之间的随机数作为level数组的大小（数组长度），它也是层的高度。
  - 每个层都带有两个属性：前进指针和跨度。
    - 前进指针：level[i].forward属性，它指向跳表的表尾，用于访问位于表尾方向的其他节点。
    - 跨度：level[i].span属性，层的跨度，它记录了前进指针所指向节点和当前节点的距离。当程序从表头向表尾进行遍历时，访问会沿着层的前进指针进行。两个节点之间的跨度越大，它们相距越远。如果跨度为0，则表示前进指针指向NULL，它们没有连向任何节点。
使用前进指针来实现遍历操作。使用跨度来计算排位（rank），就是在查找某个节点时，将访问过的所有层的跨度累计起来（累加和），这个结果数值就是目标节点在跳表中的排位。
- backward属性：节点的后退指针，指向前节点的前一个节点，它在程序从表尾向表头遍历节点时使用。每个节点只有一个后退指针，与前进指针有所不同，因此每次只能后退到前一个节点。 
- score属性：该属性表示跳表节点的分数，是一个双精度类型的浮点数。在跳表中，节点按各自所保存的分值从小到大进行排序。 
- obj属性：该属性表示跳表中节点所保存的成员对象，是一个指针，它指向一个保存着SDS值的字符串对象。 
跳表的表头节点和其他节点的结构是一样的，比如，后退指针、分数和成员对象在表头节点中同样存在，但是表头的这些属性都不会被用到。在一个跳表中，各个节点所保存的成员对象必须是唯一的，但是多个节点的分数值却可以是相同的。如果多个节点的分数值是相同的，那么这些节点将会按照成员对象在字典序中的大小来进行排序，成员对象的字典序比较小的节点会排在跳表的前面，而成员对象的字典序比较大的节点会排在跳表的后面。 
多个跳表节点组成跳表。Redis使用zskiplist结构来管理这些跳表节点，使得程序可以方便地对整个跳表进行处理，比如，快速访问跳表节点、快速获取跳表中的节点数量等。 
zskiplist结构的定义如下：
```c
typedef struct zskiplist {
    //跳表的表头节点和表尾节点
    struct zskiplistNode *header, *tail;
    //跳表中的节点数量
    unsigned long length;
    //跳表中层数最大的节点层数
    int level;
} zskiplist;
```
属性说明如下。
- header指针用于指向跳表的表头节点，而tail指针用于指向跳表的表尾节点。通过header和tail指针，程序可以快速查找跳表中的任何一个节点。 
- length属性：该属性是一个无符号的长整型（long）数值，用于记录跳表中的节点数量。 
- level属性：该属性是一个int类型的数值，用于获取跳表中层数最大的节点层数，不计算表头节点的层高。 

#### 7.7.2 跳表API函数
跳表API函数列举如下。
- zslCreate函数：该函数用于创建一个新的跳表。 
- zslInsert函数：该函数用于将包含给定成员和分数值的新节点插入跳表中。 
- zslDelete函数：该函数用于删除跳表中指定成员和分数值的节点。 
- zslFree函数：该函数用于释放指定的跳表，包含跳表中的所有节点，也就是清空跳表。 
- zslGetRank函数：该函数用于获取指定成员和分数值的节点在跳表中的排位。 
- zslGetElementByRank函数：该函数用于获取跳表在指定排位上的节点。 
- zslFirstInRange函数：该函数用于返回跳表中第一个符合给定一个分数值范围的节点。 
- zslLastInRange函数：该函数用于返回跳表中最后一个符合给定一个分数值范围的节点。 
- zslIsInRange函数：给定一个分数值范围（range），如11~19，如果跳表中有至少一个节点的分数值在这个范围内，就返回1；否则返回0。换句话说，如果zslIsInRange函数返回1，则表示跳表中至少有一个节点符合给定范围的分数值。 
- zslDeleteRangeByScore函数：该函数用于给定一个分数值（score）范围，删除跳表中所有在这个分数值范围内的节点。 
- zslDeleteRangeByRank函数：该函数用于给定一个排位（rank）范围，删除跳表中所有在这个排位范围内的节点。 

### 7.8 Redis中的对象
前面几节介绍了Redis用到的几类数据结构，如简单动态字符串、链表、压缩列表、快速列表、字典、整数集合及跳表等。在Redis中，并没有直接使用这些数据结构来实现键值对存储数据库，而是在这些数据结构的基础上，创建了一个对象系统，这个对象系统中包含了Redis的5种数据对象，分别是字符串对象、列表对象、哈希对象、集合对象及有序集合对象。Redis的每种数据对象都用到了前面介绍的至少一种数据结构。
通过这5种不同的数据对象，我们可以针对不同的使用场景，来为对象设置多种不同的数据结构的实现。Redis在执行命令之前，一个对象是否可以执行给定的命令是根据对象的类型来判断的；Redis的对象系统实现了基于引用计数技术的内存回收机制，当某个对象不再被程序使用的时候，这个对象所占用的内存空间就会被系统自动回收。
为了节省内存空间，使得多个数据库键可以共享同一个对象，Redis通过引用计数技术实现了对象共享机制来解决这一问题。
Redis的对象带有访问时间记录信息，它用于计算数据库键的空转时间。如果服务器启动了maxmemory功能，则空转时间较大的键可能会被服务器优先删除，以此来达到优化系统的目的。

#### 7.8.1 对象类型
Redis数据库中的键和值是由对象来表示的。每当我们在数据库中创建一个键值对时，系统至少会创建两个对象：一个对象用作键值对中的键（键对象）；另一个对象用作键值对中的值（值对象）。比如，执行以下命令：
```
127.0.0.1:6379> SET username "liuhefei"
OK
```
上述命令将会创建两个对象：一个对象是键值对中的键对象，也就是包含了字符串值“username”的对象；而另一个对象是键值对中的值对象，也就是包含了字符串值“liuhefei”的对象。
对象结构体定义如下：
```c
typedef struct redisObject {
    //对象类型
    unsigned type:4;
    //对象编码
    unsigned encoding:4;
    //指向底层实现数据结构的指针
    void *ptr;
    // ...
} robj;
```
这里的对象结构体中省略了部分属性的定义，只定义了结构体中与保存数据有关的3个属性。
- type属性：该属性用于记录对象的类型。Redis中对象的类型如下。
  - REDIS_STRING：字符串对象。
  - REDIS_LIST：列表对象。
  - REDIS_HASH：哈希对象。
  - REDIS_SET：集合对象。
  - REDIS_ZSET：有序集合对象。
Redis数据库中保存的键值对中的键总是一个字符串对象，而值可以是字符串对象、列表对象、哈希对象、集合对象及有序集合对象中的任意一种。

当一个数据库键为“字符串键”时，这个键所对应的值是字符串对象。

当一个数据库键为“列表键”时，这个键所对应的值是列表对象。

通常可以使用Redis的TYPE命令来查看一个数据库键对应的值对象的类型。


使用TYPE命令查看数据库键对应的值对象的类型，操作如下：
```
127.0.0.1:6379> SELECT 1 #切换到1号数据库
OK
127.0.0.1:6379[1]> SET username "liuhefei" #设置字符串键值对
OK
127.0.0.1:6379[1]> TYPE username
string #字符串类型
127.0.0.1:6379[1]> RPUSH numbers 6 8 2 4 9 #添加多个整数到列表numbers的表尾
(integer) 5
127.0.0.1:6379[1]> TYPE numbers
list #列表类型
127.0.0.1:6379[1]> HMSET color R "red" G "green" B "blue" #添加多个键值对到哈希表color中
OK
127.0.0.1:6379[1]> TYPE color
hash #哈希类型
127.0.0.1:6379[1]> SADD citys beijing shanghai wuhan shenzhen #添加多个城市到集合citys中
(integer) 4
127.0.0.1:6379[1]> TYPE citys
set #集合类型
127.0.0.1:6379[1]> ZADD score 98 "xiaoming" 86 "zhangsan" 100 "lisi" #添加多个学生与成绩到有序集合score中
(integer) 3
127.0.0.1:6379[1]> TYPE score
zset #有序集合类型
```
使用TYPE命令查看数据库键对应的值对象的类型总结为表7.2。 （此处未列出表7.2具体内容，原文未显示 ） 
**表7.2 TYPE命令输出不同值对象的类型**
|对象类型|对象type属性的值|TYPE命令的输出|
| ---- | ---- | ---- |
|字符串对象|REDIS_STRING|string|
|列表对象|REDIS_LIST|list|
|哈希对象|REDIS_HASH|hash|
|集合对象|REDIS_SET|set|
|有序集合对象|REDIS_ZSET|zset|

- encoding属性：该属性记录了对象使用何种编码方式，也就是这个对象底层使用了什么数据结构来实现。
- ptr属性：该属性是一个指针，用于指向对象的底层实现数据结构，而这些数据结构由对象的encoding属性决定。

对象的编码常量及对应的底层数据结构如下。
- 当编码常量为REDIS_ENCODING_INT时，对应的底层数据结构是long类型的整数。
- 当编码常量为REDIS_ENCODING_EMBSTR时，对应的底层数据结构是采用embstr编码的简单动态字符串。
- 当编码常量为REDIS_ENCODING_RAW时，对应的底层数据结构是简单动态字符串。
- 当编码常量为REDIS_ENCODING_HT时，对应的底层数据结构是字典。
- 当编码常量为REDIS_ENCODING_LINKEDLIST时，对应的底层数据结构是双向链表。
- 当编码常量为REDIS_ENCODING_ZIPLIST时，对应的底层数据结构是压缩列表。
- 当编码常量为REDIS_ENCODING_INTSET时，对应的底层数据结构是整数集合。
- 当编码常量为REDIS_ENCODING_SKIPLIST时，对应的底层数据结构是字典和跳表。

每种对象type属性的值都会对应不同的编码方式，因此对象的底层实现也就不一样。
**表7.3 不同type类型编码及对象的底层实现**
|type属性的值（类型）|编码|对象的底层实现|
| ---- | ---- | ---- |
|REDIS_STRING|REDIS_ENCODING_INT|采用整数值实现字符串对象|
|REDIS_STRING|REDIS_ENCODING_EMBSTR|采用embstr编码的简单动态字符串实现字符串对象|
|REDIS_STRING|REDIS_ENCODING_RAW|采用简单动态字符串实现字符串对象|
|REDIS_LIST|REDIS_ENCODING_ZIPLIST|采用压缩列表实现列表对象|
|REDIS_LIST|REDIS_ENCODING_QUICKLIST|采用快速列表实现列表对象|
|REDIS_LIST|REDIS_ENCODING_LINKEDLIST|采用双向链表实现列表对象|
|REDIS_HASH|REDIS_ENCODING_ZIPLIST|采用压缩列表实现哈希对象|
|REDIS_HASH|REDIS_ENCODING_HT|采用字典实现哈希对象|
|REDIS_SET|REDIS_ENCODING_INTSET|采用整数集合实现集合对象|
|REDIS_SET|REDIS_ENCODING_HT|采用字典实现集合对象|
|REDIS_ZSET|REDIS_ENCODING_ZIPLIST|采用压缩列表实现有序集合对象|
|REDIS_ZSET|REDIS_ENCODING_SKIPLIST|采用跳表和字典实现有序集合对象|

使用命令OBJECT ENCODING key可以查看一个数据库键对应的值对象的编码，操作如下：
```
127.0.0.1:6379> SELECT 2 #切换到2号数据库
OK
127.0.0.1:6379[2]> SET message "good luck!" #设置一条短消息
OK
127.0.0.1:6379[2]> OBJECT ENCODING message
"embstr" #采用embstr编码的简单动态字符串
127.0.0.1:6379[2]> SET article "Learning is easy, learning hard, learning cherishing." #设置一个长字符串
OK
127.0.0.1:6379[2]> OBJECT ENCODING article
"raw" #简单动态字符串
127.0.0.1:6379[2]> LPUSH numbers 78 89 90 100 70 60 76 80 #将多个整数添加到列表numbers中
(integer) 8
127.0.0.1:6379[2]> OBJECT ENCODING numbers
"quicklist" #快速列表
127.0.0.1:6379[2]> HMSET user userName "liuhefei" passWord "123456" age 24 #将一条用户信息添加到哈希表user中
OK
127.0.0.1:6379[2]> OBJECT ENCODING user
"ziplist" #压缩列表
127.0.0.1:6379[2]> SADD nums 1 4 8 16 32 64 #将多个整数值添加到集合nums中
(integer) 6
127.0.0.1:6379[2]> OBJECT ENCODING nums
"intset" #整数集合
127.0.0.1:6379[2]> SADD news "There will be rain tomorrow" #添加一个字符串到集合news中
(integer) 1
127.0.0.1:6379[2]> OBJECT ENCODING news
"hashtable" #字典
127.0.0.1:6379[2]> ZADD score 70 "lisi" 80 "zhangsan" 90 "wangwu" 100 "tianqi" #添加多个学生与分数到有序集合score中
(integer) 4
127.0.0.1:6379[2]> OBJECT ENCODING score
"ziplist" #压缩列表
```
以上操作列举出了不同对象的编码常量所对应的OBJECT ENCODING命令的输出形式，总结如表7.4所示。
**表7.4 不同对象的编码常量所对应的OBJECT ENCODING命令的输出形式**
|对象的底层数据结构|编码常量|OBJECT ENCODING命令的输出|
| ---- | ---- | ---- |
|采用embstr编码的简单动态字符串（SDS）|REDIS_ENCODING_EMBSTR|"embstr"|
|简单动态字符串|REDIS_ENCODING_RAW|"raw"|
|双向链表|REDIS_ENCODING_LINKEDLIST|"linkedlist"|
|字典|REDIS_ENCODING_HT|"hashtable"|
|压缩列表|REDIS_ENCODING_ZIPLIST|"ziplist"|
|快速列表|REDIS_ENCODING_QUICKLIST|"quicklist"|
|整数集合|REDIS_ENCODING_INTSET|"intset"|
|跳表和字典|REDIS_ENCODING_SKIPLIST|"skiplist"|

#### 7.8.2 对象的编码方式
Redis有5种数据类型，每种数据类型都有对应的对象，具体有字符串对象、哈希对象、列表对象、集合对象及有序集合对象。每种对象都有其不同的编码方式，以适应不同的应用场景，同时提高了Redis的灵活性和效率。
下面列举出每种对象可能使用的编码方式。
- 字符串对象的编码方式可能是int、raw或embstr。
- 哈希对象的编码方式可能是ziplist或hashtable。 
- 列表对象的编码方式可能是ziplist、quicklist或linkedlist。 
- 集合对象的编码方式可能是intset或hashtable。 
- 有序集合对象的编码方式可能是ziplist或skiplist。 

每种对象都有多种编码方式，那么在什么时候使用何种编码方式呢？下面我们逐一介绍。
1. **字符串对象**
   - 如果字符串对象保存的是一个long类型的整数值，那么这个字符串对象将会把这个整数值保存到字符串对象结构的ptr属性里，同时设置为int编码方式。
   - 如果字符串对象保存的是一个长度超过32字节的字符串值，那么这个字符串对象将会使用简单动态字符串来保存这个字符串值，同时设置为raw编码方式。
   - 如果字符串对象保存的是一个长度小于或等于32字节的字符串值（短字符串），那么这个字符串对象将会使用embstr编码方式来保存这个字符串值。
字符串对象的int和embstr编码在满足一定条件的情况下，会转化为raw编码。
2. **哈希对象**
   - 采用压缩列表作为底层实现了ziplist编码的哈希对象，每当要将新的键值对添加到列表中时，程序会将键值对中的键对象和值对象依次保存到压缩列表的表尾。
   - 采用字典作为底层实现了hashtable编码的哈希对象，它的每个键值对都使用一个字典键值对保存。字典中的每个键和值都是一个字符串对象；字典中的键保存键值对中的键，字典中的值保存键值对中的值。
如果哈希对象同时满足以下两个条件：
   - 哈希对象保存的所有键值对中的键和值的字符串长度不超过64字节。
   - 哈希对象保存的键值对的个数在512个之内。
则哈希对象将会使用ziplist编码方式。而如果哈希对象不满足上述条件，则将会使用hashtable作为哈希对象的编码方式。
3. **列表对象**
   - 采用ziplist编码的列表对象使用压缩列表作为其底层实现，每个列表元素都保存在一个压缩列表节点中。
   - 采用quicklist编码的列表对象使用快速列表作为其底层实现，每个快速列表的节点又是一个压缩列表。
   - 采用linkedlist编码的列表对象在底层使用双向链表实现，每个双向链表的节点都保存了一个字符串对象。
如果列表对象满足以下条件，列表对象就会使用ziplist编码方式。
   - 列表对象保存的所有字符串元素的长度都小于64字节。
   - 列表对象保存的元素个数少于512个。
如果列表对象不能满足上述条件，则将会使用linkedlist编码方式。
4. **集合对象**
   - 采用整数集合作为底层实现了intset编码的集合对象，这个集合对象所包含的所有元素都会被保存到这个整数集合中。
   - 采用字典作为底层实现了hashtable编码的集合对象，字典中的每个键都是一个字符串对象，每个字符串对象都包含了集合元素，而字典的值被全部设置为NULL。
如果集合对象满足以下两个条件，则将会使用intset编码方式。
   - 集合对象中的所有元素都是整数值。
   - 集合对象的所有元素个数之和在512个之内。
如果集合对象不满足上述条件，则将会使用hashtable编码方式。
5. **有序集合对象**
   - 采用压缩列表作为底层实现了ziplist编码的有序集合对象，每个有序集合的元素使用两个相连的压缩列表节点来保存，第一个压缩列表的节点保存有序集合元素的成员（member），第二个压缩列表的节点保存有序集合元素的分数值（score）。压缩列表内的集合元素会根据分数值的大小，按从小到大的顺序进行排序，分数值较小的元素会被放置在靠近表头的一端，而分数值较大的元素会被放置在靠近表尾的一端。
   - 采用zset结构作为底层实现了skiplist编码的有序集合对象，一个zset结构同时包含一个跳表和一个字典。
如果有序集合对象同时满足以下两个条件：
   - 有序集合所保存的元素个数之和在128个之内。
   - 有序集合保存的所有元素的长度小于64字节。
则有序集合对象使用ziplist编码方式。
如果有序集合对象不满足上述条件，则使用skiplist编码方式。

Redis底层数据结构对应的源码文件如表7.5所示。
**表7.5 Redis底层数据结构对应的源码文件**
|Redis底层数据结构|源码文件|
| ---- | ---- |
|简单动态字符串（SDS）|sds.c和sds.h|
|链表|adlist.c和adlist.h|
|压缩列表（ziplist）|ziplist.c和ziplist.h|
|快速列表（quicklist）|quicklist.c和quicklist.h|
|字典|dict.c和dict.h|
|整数集合（intset）|intset.c和intset.h|
|跳表（skiplist）|t_zset.c和redis.h|
|对象系统（redisObject）|object.c和server.h| 
