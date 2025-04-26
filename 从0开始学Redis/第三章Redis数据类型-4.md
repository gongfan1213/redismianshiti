### 3. ZLEXCOUNT命令：获取在指定区间内的元素数量
命令格式：
```
ZLEXCOUNT key min max
```


ZLEXCOUNT命令用于获取有序集合key中介于min和max范围内的元素数量，这个有序集合key中的所有元素的score值都相等。

参数min和max是一个区间，区间一般使用“(”或“[”表示，其中，“(”表示开区间，“(”指定的值不会被包含在范围之内；“[”表示闭区间，“[”指定的值会被包含在范围之内。另外，特殊值+和 - 在参数min和max中具有特殊

含义，其中，+表示正无穷，- 表示负无穷。我们向一个元素分数相同的有序集合发送命令ZLEXCOUNT <zset> - +，将会返回这个有序集合中的所有元素。



返回值：ZLEXCOUNT命令成功执行后，返回一个整数值，表示在指定范围内的元素数量。

获取有序集合citys-GDP1在指定区间内的元素数量，操作如下：


```
127.0.0.1:6379[4]> ZADD citys-GDP1 12000 '北京' 12000 '上海' 12000 '广州' 12000 '深圳' 12000 '武汉' 12000 '昆明' #有序集合citys-GDP1中的所有元素分数值都相等
6
127.0.0.1:6379[4]> ZLEXCOUNT citys-GDP1 - + #获取有序集合中的所有元素数量
6
127.0.0.1:6379[4]> ZLEXCOUNT citys-GDP1 (上海 [深圳
5
127.0.0.1:6379[4]> ZLEXCOUNT citys-GDP1 [上海 [上海
1
127.0.0.1:6379[4]> ZLEXCOUNT citys-GDP1 (上海 (昆明
2
```

### 4. ZRANGE命令：获取在指定区间内的元素（升序）
命令格式：
```
ZRANGE key start stop [WITHSCORES]
```



ZRANGE命令用于返回有序集合key中指定区间内的元素。返回的元素按照score值从小到大的顺序排序。具有相同score值的元素会按照字典序排序。

参数start和stop的默认值为0。0表示有序集合key中的第一个元素，1表示有序集合key中的第二个元素，以此类推。使用-1表示有序集合key中的最后一个元素，使用-2表示有序集合key中的倒数第二个元素，以此类推。

超出有序集合的下标不会引起错误。

当start的值大于有序集合key的最大下标，或者start大于stop时，ZRANGE命令什么也不做，只是简单地返回一个空列表。当stop的值大于有序集合key的最大下标时，Redis会将这个stop的值作为有序集合的新下标。

可以使用WITHSCORES选项来实现同时返回集合元素和这些元素所对应的score值，返回的格式是：value1,score1, ... ,valueN,scoreN。返回的元素的数据类型可能会很复杂，如元组、数组等。

返回值：ZRANGE命令成功执行后，会返回指定区间内带有score值的列表元素集合。

获取有序集合citys-GDP在指定区间内的元素，操作如下：

```
127.0.0.1:6379[4]> ZRANGE citys-GDP 0 -1
桂林
昆明
贵阳
武汉
南京
深圳
广州
上海
北京
127.0.0.1:6379[4]> ZRANGE citys-GDP 0 -1 WITHSCORES
桂林
2451.5500000000002
昆明
7726.1899999999996
贵阳
8633
武汉
9854
南京
10246
深圳
11562
广州
13240
上海
15754
北京
15892
127.0.0.1:6379[4]> ZRANGE citys-GDP 4 -1
南京
深圳
广州
上海
北京
```

### 5. ZREVRANGE命令：获取在指定区间内的元素（降序）

命令格式：
```js
ZREVRANGE key start stop [WITHSCORES]
```

ZREVRANGE命令用于返回有序集合key中指定区间内的元素。返回的元素按照score值从大到小的顺序排序。如果有相同score值的元素，则按照字典序的逆序排序。

使用WITHSCORES选项来返回元素的score值。

ZREVRANGE命令与ZRANGE命令相反，ZRANGE命令按照score值从小到大的顺序返回有序集合元素。

返回值：ZREVRANGE命令成功执行后，返回指定区间内的有序集合元素。

获取有序集合citys-GDP在指定区间内的元素，操作如下：

```js
127.0.0.1:6379[4]> ZREVRANGE citys-GDP 0 -1 WITHSCORES

北京
15892
上海
15754
广州
13240
深圳
11562
南京
10246
武汉
9854
贵阳
8633
昆明
7726.1899999999996
桂林
2451.5500000000002
```
### 6. ZSCORE命令：获取元素的分数值

命令格式：
```js
ZSCORE key member
```

ZSCORE命令用于返回有序集合key中member元素的score值。

如果有序集合key中不存在member元素，或者key不存在，则返回nil。

返回值：ZSCORE命令成功执行后，以字符串的形式返回member元素的score值。

获取有序集合citys-GDP中元素的分数值，操作如下：

```js
127.0.0.1:6379[4]> ZSCORE citys-GDP '北京'
15892
127.0.0.1:6379[4]> ZSCORE citys-GDP '上海' '武汉'
ERR wrong number of arguments for 'zscore' command
127.0.0.1:6379[4]> ZSCORE citys-GDP '上海'
15754
127.0.0.1:6379[4]> ZSCORE citys-GDP '昆明'
7726.1899999999996
```

### 7. ZRANGEBYLEX命令：获取集合在指定范围内的元素
命令格式：
```js
ZRANGEBYLEX key min max [LIMIT offset count]
```

ZRANGEBYLEX命令用于返回有序集合key中，元素score值介于min和max之间的元素，这个有序集合key中的所有元素具有相同的score值，它们按照字典序排序。如果有序集合key中的元素对应的score值不同，则在执行该命令后，返回的结果是未指定的（unspecified）。

可选的LIMIT offset count参数用于获取指定范围内的匹配元素。此时，需要注意，如果offset参数的值非常大，那么该命令在返回结果之前，需要先遍历到offset所指定的位置。


参数min和max是一个区间，区间一般使用“(”或“[”表示，其中，“(”表示开区间，“(”指定的值不会被包含在范围之内；“[”表示闭区间，“[”指定的值会被包含在范围之内。另外，特殊值+和 - 在参数min和max中具有特殊含义，其中，+表示正无穷，- 表示负无穷。我们向一个元素分数相同的有序集合发送命令ZRANGEBYLEX <zset> - +，将会返回这个有序集合中的所有元素。

返回值：ZRANGEBYLEX命令成功执行后，将会返回有序集合在指定范围内的元素。

按指定的分数区间返回有序集合citys-GDP1中的元素，前提是有序集合citys-GDP1元素的分数值都相等，操作如下：

```js
127.0.0.1:6379[4]> ZRANGEBYLEX citys-GDP1 - +
上海
北京
广州
昆明
武汉
深圳
127.0.0.1:6379[4]> ZLEXCOUNT citys-GDP1 (上海 (昆明
2
127.0.0.1:6379[4]> ZRANGEBYLEX citys-GDP1 (上海 +
北京
广州
昆明
武汉
深圳
127.0.0.1:6379[4]> ZRANGEBYLEX citys-GDP1 - [深圳
上海
北京
广州
昆明
武汉
深圳
127.0.0.1:6379[4]> ZRANGEBYLEX citys-GDP1 (广州 (武汉
昆明
```
### 8. ZRANGEBYSCORE命令：获取在指定分数区间内的元素
命令格式：
```js
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
```
ZRANGEBYSCORE命令用于返回有序集合key中，所有score值介于min和max之 

### 8. ZRANGEBYSCORE命令：获取在指定分数区间内的元素

命令格式：

```js
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
```


ZRANGEBYSCORE命令用于返回有序集合key中，所有score值介于min和max之间（包含等于min和max）的元素。有序集合key中的元素按照score值从小到大的顺序排序。当不知道min和max参数的具体值时，可以使用-inf来
表示min值，使用+inf来表示max值。在默认情况下，min与max区间是闭区间（小于等于或大于等于），也可以在参数前面添加“(”符号来使用可选的开区间（小于或大于）。

当具有相同score值的元素时，有序集合元素会按照字典序排序。

使用WITHSCORES选项来返回元素的score值。

可选的LIMIT offset count参数用于获取指定范围内的匹配元素。如果offset参数的值非常大，那么该命令在返回结果之前，需要先遍历到offset所指定的位置。

返回值：ZRANGEBYSCORE命令成功执行后，返回在指定分数区间内的有序集合元素。

返回在指定分数区间内的有序集合citys-GDP中的元素，操作如下：

```js
127.0.0.1:6379[4]> ZRANGEBYSCORE citys-GDP 7000 12000 WITHSCORES

昆明
7726.1899999999996
贵阳
8633
武汉
9854
南京
10246
深圳
11562
127.0.0.1:6379[4]> ZRANGEBYSCORE citys-GDP 9000 +inf
武汉
南京
深圳
广州
上海
北京
127.0.0.1:6379[4]> ZRANGEBYSCORE citys-GDP 9000 +inf WITHSCORES
武汉
9854
南京
10246
深圳
11562
广州
13240
上海
15754
北京
15892
127.0.0.1:6379[4]> ZRANGEBYSCORE citys-GDP (11000 (13240 WITHSCORES
127.0.0.1:6379[4]> ZRANGEBYSCORE citys-GDP (8000 (13240
```

### 9. ZREVRANGEBYSCORE命令：获取在指定区间内的所有元素
命令格式：
```js
ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]
```

ZREVRANGEBYSCORE命令用于返回有序集合key中，score值介于max和min之间（包含等于max和min）的所有元素。返回的元素按照score值从大到小的顺序排序。具有相同score值的元素按照字典序的逆序排序。

使用WITHSCORES选项来同时返回有序集合元素所对应的score值。

可选的LIMIT offset count参数用于获取指定范围内的匹配元素。如果offset参数的值非常大，那么该命令在返回结果之前，需要先遍历到offset所指定的位置。

ZREVRANGEBYSCORE命令除score值按照从大到小的顺序排序之外，其他都与ZRANGEBYSCORE命令相同。

返回值：返回在指定区间内的所有有序集合元素。

获取有序集合citys-GDP在指定区间内的所有元素，操作如下：

```js
127.0.0.1:6379[4]> ZREVRANGEBYSCORE citys-GDP 8000 1100 WITHSCORES
昆明
7726.1899999999996
桂林
2451.5500000000002
127.0.0.1:6379[4]> ZREVRANGEBYSCORE citys-GDP 15000 10000 WITHSCORES
广州
13240
深圳
11562
南京
10246
127.0.0.1:6379[4]> ZREVRANGEBYSCORE citys-GDP (15000 (11000 WITHSCORES
广州
13240
深圳
11562
```

### 3.5.3 有序集合排名
#### 1. ZRANK命令：获取有序集合元素的排名
命令格式：
```js
ZRANK key member
``

ZRANK命令用于获取有序集合key中member元素的排名。其中，有序集合元素会按score值从小到大的顺序排序。排名以0为底，换句话说，就是score值最小的元素排名为0。
返回值：如果member不是有序集合key中的元素，则返回nil；反之，如果member是有序集合key中的元素，则在执行该命令后，返回member元素的排名。
获取有序集合citys-GDP中元素的排名，操作如下：

```js
127.0.0.1:6379[4]> ZRANGE citys-GDP 0 -1
桂林
昆明
贵阳
武汉
南京
深圳
广州
上海
北京
127.0.0.1:6379[4]> ZRANK citys-GDP '桂林'
0
127.0.0.1:6379[4]> ZRANK citys-GDP '上海'
7
127.0.0.1:6379[4]> ZRANK citys-GDP '南京'
4
```

#### 2. ZREVRANK命令：获取有序集合元素的倒序排名
命令格式：
```js
ZREVRANK key member
```
ZREVRANK命令用于返回有序集合key中member元素的排名，其中，有序集合元素按照score值从大到小的顺序排序。排名以0为底，换句话说，就是score值最大的成员排名为0。
返回值：如果有序集合key中存在member元素，则返回member元素的排名；如果有序集合key中不存在member元素，则返回nil。
获取有序集合citys-GDP中元素的倒序排名，操作如下：
```js
127.0.0.1:6379[4]> ZREVRANGE citys-GDP 0 -1
北京
上海
广州
深圳
南京
武汉
贵阳
昆明
桂林
127.0.0.1:6379[4]> ZREVRANK citys-GDP '北京'
0
127.0.0.1:6379[4]> ZREVRANK citys-GDP '上海'
1
127.0.0.1:6379[4]> ZREVRANK citys-GDP '桂林'
8
```
### 3.5.4 有序集合运算
#### 1. ZINTERSTORE命令：保存多个有序集合的交集
命令格式：
```js
ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM | MIN | MAX]
```

ZINTERSTORE命令用于计算给定的一个或多个有序集合key的交集，其中给定key的数量必须和numkeys相等，并将该交集存储到destination中。

在默认情况下，交集（结果集）中的某个元素的score值是所有给定有序集合中该元素的score值之和。

返回值：ZINTERSTORE命令成功执行后，返回保存到destination结果集中的元素个数。

计算有序集合citys-GDP3、citys-GDP4的交集，并存入有序集合citys-GDP5中，操作如下：

```js
127.0.0.1:6379[4]> ZADD citys-GDP3 6783 '北京' 5000 '上海' 5439 '广州' 7012 '深圳' 9423 '武汉'
5
127.0.0.1:6379[4]> ZADD citys-GDP4 7651 '北京' 5000 '上海' 6130 '广州' 3791 '深圳' 8759 '昆明'
5
127.0.0.1:6379[4]> ZINTERSTORE citys-GDP5 2 citys-GDP3 citys-GDP4 #有序集合交集运算
4
127.0.0.1:6379[4]> ZRANGE citys-GDP5 0 -1 WITHSCORES
上海
10000
深圳
10803
广州
11569
北京
14434
```

#### 2. ZUNIONSTORE命令：保存多个有序集合的并集
命令格式：
```js
ZUNIONSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM | MIN | MAX]
```


ZUNIONSTORE命令用于计算给定的一个或多个有序集合key的并集，其中给定key的数量必须等于numkeys，并将计算的并集结果存入destination中。在默认情况下，这个并集结果中的某个元素的score值是所有指定集合中该元素的score值之和。

使用WEIGHTS选项来为每个给定的有序集合分别指定一个乘数，每个给定的有序集 



合中的所有元素的score值在传递给聚合函数之前都会乘以这个乘数（weight）。如果没有指定WEIGHTS选项，则乘数默认设置为1。

使用AGGREGATE选项来指定计算并集结果的聚合方式。具体如下。
- SUM：默认的聚合方式，它可以将所有有序集合中某个元素的score值之和作为结果集中该元素的score值。
- MIN：这种聚合方式可以将所有有序集合中某个元素的最小score值作为结果集中该元素的score值。 
- MAX：这种聚合方式可以将所有有序集合中某个元素的最大score值作为结果集中该元素的score值。 

返回值：ZUNIONSTORE命令成功执行后，返回保存到destination结果集中的元素数量。

计算有序集合citys-GDP6、citys-GDP7的并集，并将结果存入有序集合citys-GDP8中，操作如下：
```js
127.0.0.1:6379[4]> ZADD citys-GDP6 3415 '北京' 2500 '上海' 3201 '苏州' 2893 '杭州'
4
127.0.0.1:6379[4]> ZADD citys-GDP7 5438 '北京' 3700 '上海' 5422 '贵阳' 4391 '昆明'
4
127.0.0.1:6379[4]> ZUNIONSTORE citys-GDP8 2 citys-GDP6 citys-GDP7 WEIGHTS 4 1.5 AGGREGATE MIN #citys-GDP6 * 4, citys-GDP7 * 1.5, 并集结果取最小分数值
6
127.0.0.1:6379[4]> ZRANGE citys-GDP8 0 -1 WITHSCORES
上海
5500 #3700 * 1.5
昆明
6586.5 #4391 * 1.5
贵阳
8133 #5422 * 1.5
北京
8157 #5438 * 1.5
杭州
11572 #2893 * 4
苏州
12804 #3201 * 4
```
### 3.5.5 删除有序集合元素
#### 1. ZREM命令：删除有序集合中的多个元素
命令格式：
```js
ZREM key member [member ...]
```
ZREM命令用于删除有序集合key中的一个或多个元素，不存在的元素会被忽略。当key存在，但它不是有序集合类型时，返回一个错误。

返回值：ZREM命令成功执行后，返回被删除元素的数量，不包括被忽略的元素。

删除有序集合citys-GDP2中的多个元素，操作如下：
```js
127.0.0.1:6379[4]> ZRANGE citys-GDP2 0 -1
杭州
苏州
上海
昆明
贵阳
北京
127.0.0.1:6379[4]> ZREM citys-GDP2 '贵阳'
1
127.0.0.1:6379[4]> ZREM citys-GDP2 '贵阳'
0
127.0.0.1:6379[4]> ZREM citys-GDP2 '北京' '上海' '贵阳' '昆明'
invalid argument(s)
127.0.0.1:6379[4]> ZREM citys-GDP2 '北京' '上海' '昆明'
3
```
#### 2. ZREMRANGEBYLEX命令：删除有序集合在指定区间内的元素
命令格式：

```js
ZREMRANGEBYLEX key min max
```


ZREMRANGEBYLEX命令用于删除有序集合key中，介于min和max范围内的score值相同的所有元素。该命令的min和max参数的意义与ZRANGEBYLEX命令的min和max参数的意义相同。

返回值：ZREMRANGEBYLEX命令成功执行后，返回被删除元素的数量。

删除有序集合citys-GDP9在指定区间内的元素，有序集合citys-GDP9的元素分数值相同，操作如下：

```js
127.0.0.1:6379[4]> ZADD citys-GDP9 12000 '北京' 12000 '苏州' 12000 '杭州' 12000 '深圳' 12000 '合肥' 12000 '昆明'
6
127.0.0.1:6379[4]> ZREMRANGEBYLEX citys-GDP9 - [北京
1
127.0.0.1:6379[4]> ZREMRANGEBYLEX citys-GDP9 (昆明 +
3
127.0.0.1:6379[4]> ZRANGE citys-GDP9 0 -1
合肥
昆明
127.0.0.1:6379[4]> ZREMRANGEBYLEX citys-GDP9 - + #删除有序集合中的全部元素
2
127.0.0.1:6379[4]> ZRANGE citys-GDP9 0 -1
```
#### 3. ZREMRANGEBYRANK命令：删除有序集合在指定排名区间内的元素
命令格式：
```js
ZREMRANGEBYRANK key start stop
```

ZREMRANGEBYRANK命令用于删除有序集合key在指定排名（rank）区间内的所有元素。区间范围由下标参数start和stop给出，包含start和stop在内。下标参数start和stop的用法与命令ZRANGE中下标参数start和stop的用法相同，在此不再说明。

返回值：返回被删除元素的数量。

删除有序集合citys-GDP在指定排名区间内的元素，操作如下：
```js
127.0.0.1:6379[4]> ZRANGE citys-GDP2 0 -1
杭州
苏州
上海
昆明
贵阳
北京
127.0.0.1:6379[4]> ZREMRANGEBYRANK citys-GDP2 0 3 #删除有序集合citys-GDP2的元素排名在0~3之间的元素
4
127.0.0.1:6379[4]> ZREMRANGEBYRANK citys-GDP2 3 20 #排名区间超过了有序集合排名, 删除失败
0
127.0.0.1:6379[4]> ZRANGE citys-GDP2 0 -1
贵阳
北京
```
#### 4. ZREMRANGEBYSCORE命令：删除有序集合在指定分数区间内的元素
命令格式：
```js
ZREMRANGEBYSCORE key min max
```
ZREMRANGEBYSCORE命令用于删除有序集合key中，所有score值介于min和max之间（包含等于min或max）的元素。
返回值：ZREMRANGEBYSCORE命令成功执行后，返回被删除元素的数量。
删除有序集合citys-GDP在指定分数区间内的元素，操作如下：
```js
127.0.0.1:6379[4]> ZRANGE citys-GDP 0 -1 WITHSCORES
桂林
2451.5500000000002
昆明
7726.1899999999996
贵阳
8633
武汉
9854
南京
10246
深圳
11562
广州
13240
上海
15754
北京
15892
127.0.0.1:6379[4]> ZREMRANGEBYSCORE citys-GDP 3000 4000
0 #为0，表示在3000~4000区间内没有元素
127.0.0.1:6379[4]> ZREMRANGEBYSCORE citys-GDP 2000 4000
1
127.0.0.1:6379[4]> ZREMRANGEBYSCORE citys-GDP 8000 10000
2 #表示成功删除两个元素
127.0.0.1:6379[4]> ZRANGE citys-GDP 0 -1 WITHSCORES
昆明
7726.1899999999996
南京
10246
深圳
11562
广州
13240
上海
15754
北京
15892
```
至此，与Redis数据库的数据类型相关的命令就介绍完了。命令比较多，希望读者多动手实践，这样才能记住相关命令的用法。同时也希望读者坚持学习，提高自己的技术，在实际应用中将其发挥到极致，成就自我。 
