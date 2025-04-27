### 第10章 Redis消息订阅
本章的主题为Redis的订阅发布，我们主要围绕订阅发布模式来讲解，首先为读者介绍什么是消息订阅发布模式，然后介绍Redis消息订阅的相关知识，如何实现Redis的消息订阅功能，以及Redis队列的相关知识。

#### 10.1 消息订阅发布概述
什么是消息订阅发布模式呢？消息订阅发布模式一种常用的设计模式，它具有一对多的依赖关系，它有3个角色：主题（Topic）、订阅者（Subscriber）、发布者（Publisher）。简单来说，就是让多个订阅者对象同时监听某个发布者发布的主题对象，当这个主题对象的状态发生变化时，所有订阅者对象都会收到通知，使它们自动更新自己的状态。这里所说的主题就是一条消息内容，每条消息可以被多个订阅者订阅。发布者与订阅者具有一对多的关系，它们之间存在依赖性，订阅者必须订阅主题后才能接收到发布者发布的消息，在订阅前发布的消息，订阅者是接收不到的。这就是消息订阅发布模式，如图10.1所示。

![image](https://github.com/user-attachments/assets/3e01e01a-b6a0-4b37-ae15-c85018470f31)


**图10.1 消息订阅发布模式**
```
发布者 ----发送消息----> Pub/Sub Topic
                    |<----订阅主题---- 订阅者
                    |----发送消息----> 订阅者
                    |<----订阅主题---- 订阅者
```

Redis的消息订阅发布功能主要由PSUBSCRIBE、PUBLISH、PUBSUB、PUNSUBSCRIBE、SUBSCRIBE、UNSUBSCRIBE等命令实现。当一个客户端使用PUBLISH命令向订阅者发布消息内容时，这个客户端就是消息发布者。而当另一个或多个客户端使用SUBSCRIBE或PSUBSCRIBE命令接收消息的时候，这个客户端就是消息订阅者。通过执行SUBSCRIBE命令，客户端（消息订阅者）可以同时订阅多个频道（Channel），当有其他客户端（消息发布者）向被订阅的频道发送消息时，所有订阅这个频道的消息订阅者都会接收到这条消息。

这里所说的频道就是一个中介，因为消息发布者与消息订阅者之间存在依赖关系，为了解耦两者之间的关系，就使用了频道作为中介。消息发布者将消息发送给频道，而这个频道在接收到消息后，负责把这条消息发送给所有订阅过这个频道的消息订阅者。消息发布者不需要知道具体有多少消息订阅者，一个消息订阅者可以订阅一个或多个消息频道，并且它只能接收已订阅过的频道中的消息。同理，消息订阅者也不需要知道具体有哪些消息发布者，它们之间不存在相互关系，也不需要知道对方是否存在，这就做到了很好的解耦。

#### 10.2 消息订阅发布实现
##### 10.2.1 消息订阅发布模式命令
下面介绍几个与消息订阅发布有关的命令。
1. **PUBLISH**
消息发布者将消息发送给指定的频道，返回一个整数，表示接收到这条消息的客户端的数量。
**命令格式**：
```
PUBLISH channel message
```
2. **SUBSCRIBE**
客户端订阅指定的消息频道。一旦客户端进入订阅状态，它就不能运行除SUBSCRIBE、PSUBSCRIBE、UNSUBSCRIBE和PUNSUBSCRIBE命令之外的其他命令了。
**命令格式**：
```
SUBSCRIBE channel [channel ...]
```
**具体操作**：
- **客户端1：消息发布者发布消息**
```
127.0.0.1:6379> PUBLISH infomation "The harder you work,the luckier you get"
(integer) 0  #为0，表示没有订阅者
127.0.0.1:6379> PUBLISH infomation "who are you?"
(integer) 1  #为1，表示有一个订阅者
127.0.0.1:6379> PUBLISH infomation "where is this?"
(integer) 1
127.0.0.1:6379> PUBLISH infomation "My name is liuhefei"
(integer) 1
127.0.0.1:6379> PUBLISH infomation "I like girl"
(integer) 1
127.0.0.1:6379> PUBLISH infomation "hello redis!"
(integer) 2  #为2，表示有两个订阅者
```
- **客户端2：消息订阅者订阅消息**
```
127.0.0.1:6379> SUBSCRIBE infomation
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "infomation"
3) (integer) 1
1) "message"
2) "infomation"
3) "who are you?"
1) "message"
2) "infomation"
3) "where is this?"
1) "message"
2) "infomation"
3) "My name is liuhefei"
1) "message"
2) "infomation"
3) "I like girl"
1) "message"
2) "infomation"
3) "hello redis!"
```
- **客户端3：消息订阅者订阅消息**
```
127.0.0.1:6379> SUBSCRIBE infomation
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "infomation"
3) (integer) 1
1) "message"
2) "infomation"
3) "hello redis!"
```
3. **PSUBSCRIBE**

客户端根据指定的模式来订阅符合这个模式的频道。该命令可以重复订阅一个频道，它支持glob风格的模式。
- h?llo：可以订阅hello、hallo和hxllo频道（?表示单个任意字符）。
- h*llo：可以订阅hllo和heeeello频道（*表示多个任意字符，包括空字符）。
- h[ae]llo：可以订阅hello和hallo频道，但是不能订阅hillo频道（选择[和]之间的任意一个字符）。

如果？、*、[、]等符号不是通配符，而只是简单的字符，则需要使用“\”符号进行转义。比如，h\?llo:表示消息订阅者只能订阅h\?llo频道。

**命令格式**：
```
PSUBSCRIBE pattern [pattern ...]
```
**该命令的用法具体如下**：
- **客户端1：消息发布者给不同频道发送消息**
```
127.0.0.1:6379> PUBLISH infomation "Lucky today"
(integer) 3 #表示有3个订阅者
127.0.0.1:6379> PUBLISH infomation "Beautiful girl"
(integer) 3
127.0.0.1:6379> PUBLISH message "There is good news"
(integer) 1
127.0.0.1:6379> PUBLISH mess "If so"
(integer) 1
127.0.0.1:6379> PUBLISH info "with grief"
(integer) 1
127.0.0.1:6379> PUBLISH measy "It was a messy job"
(integer) 1
```
- **客户端2：消息订阅者同时订阅多个频道的消息**
```
127.0.0.1:6379> PSUBSCRIBE info* mess*
reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "info*"
3) (integer) 1
1) "psubscribe"
2) "mess*"
3) (integer) 2
1) "pmessage"
2) "info*"
3) "infomation"
4) "Lucky today"
1) "pmessage"
2) "mess*"
3) "message"
4) "There is good news"
1) "pmessage"
2) "info*"
3) "infomation"
4) "Beautiful girl"
1) "pmessage"
2) "mess*"
3) "mess"
4) "If so"
1) "pmessage"
2) "info*"
3) "info"
4) "with grief"
1) "pmessage"
2) "mess*"
3) "mesay"
4) "It was a messy job"
```
4. **PUNSUBSCRIBE**
   
客户端根据指定的模式退订符合该模式的所有频道。如果不指定任何模式，则默认退订所有的频道。 

### 命令格式
```
PUNSUBSCRIBE [pattern [pattern ...]]
```
### 该命令的用法具体如下
```
127.0.0.1:6379> PUNSUBSCRIBE info* #退订符合info*模式的频道
1) "punsubscribe"
2) "info*"
3) (integer) 0
127.0.0.1:6379> PUNSUBSCRIBE mess* #退订符合mess*模式的频道
1) "punsubscribe"
2) "mess*"
3) (integer) 0
127.0.0.1:6379> PUNSUBSCRIBE  #退订所有的频道
1) "punsubscribe"
2) (nil)
3) (integer) 0
```
注意：使用PUNSUBSCRIBE命令只能退订通过PSUBSCRIBE命令订阅的模式，它不会影响直接通过SUBSCRIBE命令订阅的频道，也不会影响通过PSUBSCRIBE命令订阅的模式。

### 5. UNSUBSCRIBE
客户端退订指定的频道。如果没有指定任何频道参数，那么所有已订阅的频道都会被退订。
### 命令格式
```
UNSUBSCRIBE [channel [channel ...]]
```
### 该命令的用法具体如下
```
127.0.0.1:6379> UNSUBSCRIBE info*
1) "unsubscribe"
2) "info*"
3) (integer) 0
127.0.0.1:6379> UNSUBSCRIBE mess*
1) "unsubscribe"
2) "mess*"
3) (integer) 0
127.0.0.1:6379> UNSUBSCRIBE *
1) "unsubscribe"
2) "*"
3) (integer) 0
```
注意：如果客户端是在Redis命令行（redis - cli）中进入消息订阅监听状态的，那么使用PUNSUBSCRIBE命令退订消息模式和使用UNSUBSCRIBE命令退订消息频道的操作都会执行失败，也就是消息订阅退订失败。必须通过telnet之类的工具才能在消息订阅监听状态下执行退订操作。

### 6. PUBSUB命令
PUBSUB命令是一个自检命令，可用于检查发布/订阅子系统的状态。这个命令是由几个子命令组成的，命令格式如下：
```
PUBSUB <subcommand> [argument [argument ...]]
```
- **PUBSUB CHANNELS [pattern]**：该命令用于返回服务器被订阅的有效频道。有效频道是具有一个或多个订阅者（不包括订阅模式的客户端）的发布/订阅频道。如果没有指定任何模式，也就是没有pattern参数，则默认列举出所有的有效频道。如果指定了模式，也就是设置了pattern参数，则会列举出符合这个模式规则的所有频道（使用glob风格模式 ）。这个命令的返回值是一个数组，通过遍历服务器pubsub_channels字典中的所有键，它会列出所有的有效频道，包括匹配指定模式的有效频道。时间复杂度为O(N)，N是有效频道的数量。
### 该命令的用法具体如下
- **客户端1：消息发布者向多个频道发送消息**
```
127.0.0.1:6379> PUBLISH messy "don't worry"
(integer) 3
127.0.0.1:6379> PUBLISH message "Life is a messy and tangled business"
(integer) 4
127.0.0.1:6379> PUBLISH infos "come on"
(integer) 2
127.0.0.1:6379> PUBLISH infomation "You look very nice"
(integer) 5
```
- **客户端2：使用PUBSUB命令列出当前活跃的消息频道**
```
127.0.0.1:6379> PUBSUB CHANNELS
1) "message"
2) "messy"
3) "infomation"
127.0.0.1:6379> PUBSUB CHANNELS 2
(empty list or set)
127.0.0.1:6379> PUBSUB CHANNELS *
1) "message"
2) "infomation"
```
- **PUBSUB NUMSUB [channel - 1 ... channel - N]**：该命令接收多个频道作为输入参数，用于获取指定频道的订阅者的数量。它通过在pubsub_channels字典中找到频道对应的订阅者链表，然后返回订阅者链表的长度，这个长度值就是频道订阅者的数量，它不包括订阅模式的客户端。它返回一个数组，用于列出参数指定的所有频道，以及每个频道的订阅者的数量。返回值的格式为频道、数量、频道、数量、……因此，这个列表是扁平的。返回值列出的频道排列顺序和命令调用时指定频道的排列顺序是相同的。注意，在调用这个命令时可以不指定频道，此时返回值是一个空列表。当返回0时，表示这个频道没有任何订阅者。时间复杂度为O(N)，N是命令中指定的频道数量。
### 使用PUBSUB命令获取指定频道订阅者的数量，操作如下
```
127.0.0.1:6379> PUBSUB NUMSUB infomation infos message mess messy
1) "infomation"
2) (integer) 5
3) "infos"
4) (integer) 2
5) "message"
6) (integer) 4
7) "mess"
8) (integer) 0
9) "messy"
10) (integer) 3
```
- **PUBSUB NUMPAT**：该命令用于获取模式的订阅数量（所有客户端运行PSUBSCRIBE命令的总次数）。这个子命令是通过返回pubsub_patterns链表的长度来实现的，返回的这个长度就是服务器被订阅模式的数量。注意，这个数量不是订阅模式的客户端的数量，而是所有客户端订阅的模式的总数量。它返回一个整数，表示所有客户端订阅的模式的总数量。
### 使用PUBSUB命令统计当前模式的订阅数量，操作如下
```
127.0.0.1:6379> PUBSUB NUMPAT
(integer) 6
```

#### 10.2.2 消息订阅功能之订阅频道
当一个客户端执行SUBSCRIBE命令订阅一个或多个频道时，这个客户端与被订阅的频道之间就建立了一种订阅频道关系，这个订阅频道关系将会被保存到服务器状态的pubsub_channels字典里面。这个字典也是一个键值对，字典中的键就是某个被订阅的频道，而字典中的值就是一个链表，这个链表记录了所有订阅这个频道的客户端。这个字典的示意图如图10.2所示。

![image](https://github.com/user-attachments/assets/3cd20b32-f6ab-48f4-94d5-594c5e48c710)



**图10.2 pubsub_channels字典示意图**
```
pubsub_channels
  |- infomation -> client-1 -> client-3 -> client-5
  |- infos -> client-2
  |- message
  |- mess -> client-4
```

当客户端订阅某个频道时，服务器会将这个频道与客户端在pubsub_channels字典中进行关联。这个关联操作可以分为两步，具体根据这个频道是否已经有其他订阅者来划分。
- 如果这个频道被其他订阅者订阅，那么这个订阅者链表信息一定会在pubsub_channels字典中，此时相关的程序就要把订阅这个频道的客户端添加到链表的末尾。
- 如果这个频道没有任何订阅者订阅，那么pubsub_channels字典中就不存在订阅关系，此时需要程序为这个频道在pubsub_channels字典中创建一个键，并把这个键对应的值设为空链表，最后将这个客户端添加到链表中，成为链表的第一个元素。

当客户端执行UNSUBSCRIBE命令来退订某个或某些频道时，服务器将从pubsub_channels字典中删除被退订频道与客户端之间的关联关系。程序会根据被退订频道的名称，在pubsub_channels字典中进行查找，找到这个频道后，会将与这个频道相关联的退订客户端的链表信息删除。删除退订客户端之后，频道的订阅者链表变为空链表，此时说明已经没有任何订阅者订阅这个频道了，程序接着会删除pubsub_channels字典中这个频道所对应的键。

所有订阅频道与客户端的关系都记录在服务器状态的pubsub_channels字典中。在使用PUBLISH命令将消息发送给所有订阅这个频道的订阅者时，PUBLISH命令要做的就是在pubsub_channels字典中查找到所有订阅这个频道的订阅者名单（订阅者链表），然后将消息发送给名单中的所有订阅者。这个过程就是PUBLISH命令将消息发送给频道订阅者。

### 订阅频道推送消息

1. **subscribe消息**

subscribe消息表示客户端已经成功地订阅了指定的频道。返回信息的第一个值是subscribe字符串，表示指定的频道订阅成功；第二个值是订阅成功的频道名称；第三个值是目前已经成功订阅的频道数量。

2. **unsubscribe消息**

unsubscribe消息表示客户端已经成功取消订阅指定的频道。返回信息的第一个值是unsubscribe字符串，表示已经成功退订指定的频道；第二个值是要退订的频道名称；第三个值是当前客户端订阅的频道数量。如果第三个值为0，则表示这个客户端没有订阅任何频道。

3. **message消息**

message消息表示订阅频道的客户端已经成功地收到了另一个客户端向这个频道发送的消息。返回信息的第一个值是message字符串，表示返回值的类型是消息；第二个值是发送消息的频道名称；第三个值是消息的内容。 

#### 10.2.3 消息订阅功能之订阅模式
当客户端执行PSUBSCRIBE命令订阅某个或某些消息模式时，这个客户端与被订阅的消息模式之间就建立了一种订阅模式关系，这个订阅模式关系将会被保存到服务器状态的pubsub_patterns属性里面。pubsub_patterns属性是一个链表，这个链表中的每个节点都包含着一个pubsubPattern结构体，这个结构体具有client和pattern两个属性，client属性表示订阅模式的客户端，而pattern属性表示被订阅的模式。

每当消息订阅者订阅某个或某些消息模式时，服务器总会对每个被订阅的消息模式执行下面的操作：
- 重新创建一个pubsubPattern结构体，并为这个结构体的client和pattern属性赋值，client属性被赋值为订阅模式的客户端，pattern属性被赋值为被订阅的模式。
- 将这个新建的pubsubPattern结构体添加到pubsub_patterns链表的表尾。

以上两步操作的具体过程用链表结构表示如图10.3所示。
\

![image](https://github.com/user-attachments/assets/9704205e-faeb-4cf7-bd3c-deb96701cbe2)



**图10.3 pubsub_patterns链表结构**

```
redisServer
  |- pubsub_patterns -> pubsubPattern (client: client1, pattern: "info*") -> pubsubPattern (client: client2, pattern: "mess*")
  |- pubsub_patterns -> pubsubPattern (client: client4, pattern: "hello*")
```

当客户端执行PUNSUBSCRIBE命令来退订某个或某些订阅模式时，服务器将会从pubsub_patterns链表中，根据PUNSUBSCRIBE命令指定的模式来查找与这个模式相匹配的订阅模式，然后删除pattern属性为退订模式，并且client属性为执行退订模式的客户端的pubsubPattern结构体。

服务器状态中的pubsub_patterns链表记录了所有消息订阅模式的订阅关系。在使用PUBLISH命令将消息发送给所有与消息频道模式相匹配的订阅者时，PUBLISH命令需要遍历整个pubsub_patterns链表，查找出与消息频道相匹配的模式，然后将消息发送给订阅了这些模式的客户端。这个过程就是PUBLISH命令将消息发送给模式订阅者。

如果某个客户端同时订阅了多个消息模式，或者多个消息模式和消息频道，并且这些模式都能匹配到同一条消息，那么这个客户端将会收到多条相同的消息。

### 订阅模式推送消息

1. **psubscribe消息**

psubscribe消息表示客户端已经成功订阅了指定的模式。返回信息的第一个值是psubscribe字符串；第二个值是订阅的模式名称；第三个值是客户端当前已经订阅的模式数量。

2. **punsubscribe消息**

punsubscribe消息表示客户端已经成功退订了指定的消息模式。返回信息的第一个值是punsubscribe字符串，表示成功退订了指定的模式；第二个值是想要退订的模式名称；第三个值是客户端当前已经订阅的模式数量。如果第三个值为0，则表示这个客户端没有订阅任何模式。

3. **pmessage消息**

pmessage消息表示订阅模式的客户端已经成功地收到了另一个客户端向这个模式所对应的频道发送的消息。换句话说，就是消息订阅者成功收到了消息发布者所发送的与之模式相匹配的消息。



#### 10.3 Redis消息队列
常见的消息队列有两种模式，分别是消息订阅发布模式和消息生产者/消费者模式。Redis的消息队列也是基于这两种模式实现的。下面我们来具体介绍。

##### 10.3.1 消息订阅发布模式的原理
消息发布者将消息发送到相应的频道（这里的频道也可以理解为队列），消息订阅者通过订阅这个消息频道，就能接收到消息发布者所发送的消息。只要是订阅了这个消息频道的订阅者，它们接收到的消息都是一样的，也就是消息发布者将消息发送给了每位订阅者。在前面的小节中已经详细介绍了消息订阅发布模式，这里不再多说。

![image](https://github.com/user-attachments/assets/f1b88724-d1e0-4368-979f-9315a5dd32eb)


**消息订阅发布模式原理图如图10.4所示**
```
消息发布者 -> 发布消息 -> Redis Server频道(channel) -> 订阅消息 -> 消息订阅者1
                                        |-> 订阅消息 -> 消息订阅者2
                                        |-> 订阅消息 -> 消息订阅者3
                                        |-> 订阅消息 -> 消息订阅者4
                                        |-> 订阅消息 -> 消息订阅者N
```

说明：当消息发布者发送一条消息到Redis Server后，只要消息订阅者订阅了该频道，就可以接收到这样的信息。同时，消息订阅者可以订阅不同的频道。

消息订阅发布模式的适用场景：微信公众号、订阅号的推送，新闻App的推送，商城系统的信息推送等。

##### 10.3.2 消息生产者/消费者模式的原理
消息生产者将生产的消息放入消息队列里，多个消息消费者同时监听这个消息队列，谁先抢到消息，谁就从消息队列中取走消息去处理，即对于每条消息，只能被最多一个消费者消费。消息订阅发布模式不需要抢夺消息，每个订阅者得到的消息都是一样的。而消息生产者/消费者模式是一种“抢”的模式，就是消息生产者每生产并发送一条消息，只有一个消息消费者可以获得消息，谁的网速快、人品好，谁就可以获得那条消息。


![image](https://github.com/user-attachments/assets/5a0dfccd-0f63-4fb5-8eca-6bd5b56310b6)


**消息生产者/消费者模式原理图如图10.5所示**
```
消息生产者 -> 发布消息 -> Redis Server频道(消息队列) -> 开始抢夺消息 -> 消息消费者1
                                                         |-> 消息消费者2
                                                         |-> 消息消费者3
                                                         |-> 消息消费者4
                                                         |-> 消息消费者N
                                                         最终只有一个消费者抢到
```

说明：消息生产者/消费者模式也有多个消费者来消费，但是只能有一个消费者可以获得消息，其他消息消费者就只能继续监听消息队列，等待下一次抢消息。

消息生产者/消费者模式的适用场景：用户系统登录注册接收短信注册码、登录码，订单系统下单成功的短信，抢红包等。

以上就是Redis的消息订阅与消息队列的相关知识，在后面的实战章节中，我们还会给出相应的例子，期待大家继续学习。 
