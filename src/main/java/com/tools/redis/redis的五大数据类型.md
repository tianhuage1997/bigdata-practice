## redis五种数据类型

### 字符串（string）

**字符串有哪些常用的命令？**

| op | 注释 |
|---|---|
| APPEND | 将value值追加到给定的key当前对应的value的末尾 |
| GETRANGE | 获取一个给定范围内的字符串 |
| SETRANGE | 将指定的位置开始设定为给定值 |
| GETBIT | 将字符串看成二进制串，返回偏移量在子串中所对应的值 |
| SETBIT | 将字符串看成二进制串，设定偏移量对应子串中的位置为给定值 |
| BITCOUNT | 统计二进制子串中位置为1的数量，可选择指定的区间 |
| BITSTOP | 可以对二进制子串进行逻辑运算，并将结果保存到新的key-value中 |

**实现原理**

底层由redis实现的简单动态字符串(SDS)实现的

相较于c语言的字符串的有几个优点：
- 获取字符串的长度的时间复杂度为O(1)
- API安全不会造成缓冲区溢出
- 修改字符串时最多需要n次内存分配
- 可以同时保存文本和二进制数
- 可以使用c中原来的库函数

支持存储三种类型的值：
- 字符串
- 整数
- 浮点数

**redis在字符串方面有哪些不同于其他数据库？**

很多键值数据库只能将数据存储为普通的字符串，并且不提供字符串处理操作，有一些数据库虽
然支持简单的追加，但却不可以像redis一样对字符串的子串进行读写（GETRANGE）。

**redis的字符串是如何保存整数和浮点数的？**

redis字符串底层使用的也是字符串数组，所以在保存时可以使用整数和浮点数，并且他会自动
识别出你保存的是整数还是浮点数，还是字符串，如果是浮点数整数，他将会支持使用自增或
自减等操作。

### 列表（list）

**列表的常用命令**

| op | 注释 |
|---|---|
| RPUSH | 在存储在键的列表尾部插入所有指定的值。如果键不存在，则在执行推送操作之前将其创建为空列表。 |
| LPUSH | 在存储在键的列表头部插入所有指定的值。如果键不存在，则在执行推送操作之前将其创建为空列表。 |
| LPOP | 从列表的头部开始推出，非阻塞的 |
| RPOP | 从列表的头部开始推出，非阻塞的 |
| RPOPLPUSH | 原子地返回并删除存储在源位置的列表的最后一个元素（尾），并将该元素推送到存储在目标位置的列表的第一个元素（头） |
| LINSERT | 尾部插入 |
| LINDEX | 获取到对应索引的value值，和普通链表一样，索引从0开始 |
| BLPOP、BRPOP |一个阻塞列表pop。lpop和rpop的阻塞版本，因为当没有元素从任何给定的列表中弹出时，它会阻塞连接。按照给定的顺序检查给定的键。|
| BRPOPLPUSH | 阻塞的RPOPLPUSH实现，当列表中不为空时，其功能与RPOPLPUSH完全一样，但当为空时则会阻塞，直到其他的客户端将数据放入进来（可使用无限期阻塞），或者超时才会返回。 |
| LLEN | 获取当前列表的长度 |

**~~在redis3.2之前的版本list实现~~**

列底层使用了压缩列表和双向链表来实现的，在列表中对象较少时，会使用压缩列表，随着包含
的对象越来越多时，
将会逐渐转换为性能等方面更好的更适合处理大量元素的双端链表（关于压缩列表和双端链表可
查阅《redis的设计与实现》）。
```
redis:6379> RPUSH zip a b c de 12 23 45 "dd"
(integer) 8
redis:6379> OBJECT encoding zip
"ziplist"
```
但从redis3.2开始将不会在看当这样的返回。

**在redis3.2之后的版本list实现**

列表底层使用了一种数据结构实现[quick list][1]，`quicklist`是一个双向链表，不过这
个双向链表的节点使用的则是`ziplist`，如果了解过`ziplist`那将会知道，它是一个内存
紧凑的数据结构，其中的每一个数据项前后相邻，并且能够维持数据项的先后顺序。

**为什么要使用quicklist这种数据结构**

- 双向链表便于在表的两端进行push和pop操作，但是它的内存开销比较大。首先，它在每个节点
上除了要保存数据之外，还要额外保存两个指针；其次，双向链表的各个节点是单独的内存块，
地址不连续，节点多了容易产生内存碎片。
- ziplist由于是一整块连续内存，所以存储效率很高。但是，它不利于修改操作，每次数据变动
都会引发一次内存的realloc。特别是当ziplist长度很长的时候，一次realloc可能会导致大
批量的数据拷贝，进一步降低性能。

**一个quicklist节点包含多长的ziplist才能在空间和时间上达到最优？**

- 每个quicklist节点上的ziplist越短，则内存碎片越多。内存碎片多了，有可能在内存中产
生很多无法被利用的小碎片，从而降低存储效率。这种情况的极端是每个quicklist节点上的
ziplist只包含一个数据项，这就蜕化成一个普通的双向链表了。
- 每个quicklist节点上的ziplist越长，则为ziplist分配大块连续内存空间的难度就越大。
有可能出现内存里有很多小块的空闲空间（它们加起来很多），但却找不到一块足够大的空闲空间
分配给ziplist的情况。这同样会降低存储效率。这种情况的极端是整个quicklist只有一个节点，
所有的数据项都分配在这仅有的一个节点的ziplist里面。这其实蜕化成一个ziplist了。

由此可见，每个quicklist节点的ziplist要保持多长，这可能要等到具体的使用场景才能够决定。
`list-max-ziplist-size`可以进行ziplist的size配置。当列表很长时，可以使用
`list-compress-depth`进行中间段压缩。

### 散列（hash）

**常用命令**

`OP hash field value`

| op | 注释 |
|---|---|
| HSET | 设置哈希表的值不存在则创建，存在覆盖，不存在时写入成功返回1，存在时覆盖成功时返回0 
| HSETNX | 当哈希表的filed不存在时创建，field存在时放弃操作，当hash表不存在时则创建hash表，再次执行命令HSETNX，成功创建filed返回1，放弃返回0 
| HGET | 根据给定的哈希表和给定的filed查询出value值 
| HEXISTS | 检查给定的filed是否存在于哈希表中 
| HDEL | 删除指定filed 
| HLEN | 哈希表的长度，就是filed的数量 
| HSTRLEN | 返回哈希表 key 中， 与给定域 field 相关联的值的字符串长度（string length）。 
| HINCRBY | filed值自增 
| HMSET | 同HSET，支持多个value 
| HMGET | 同HGET，支持多个value 
| HKEYS | 获取指定filed的所有key 
| HVALS | 获取指定filed的所有value 
| HGETALL | 同时获取filed下所有的key-value 

**实现方式**

散列的底层提供了两种实现方式，`ziplist`和`hashtable`，在一定条件下两种方式会发生相
互转换，当散列表较小时，默认使用的时`ziplist`，当一个filed存储过多的key-value时会
转而使用`hashtable`。

**ziplist如何实现hash**

ziplist使用entry保存每一对键值对，当有新的加入进来时，key会先放到压缩列表的的尾部，然后再
将value放到尾部，保证每一个key和value时紧挨着的，这样先放入的键值对会存在压缩列表的头部，
后方进来的会保持在尾部。

**hashtable如何实现的hash**

hashtable实现hash使用的时字典进行保存键值对，字典的键保存键值对的键，值保存键值对的
值。字典中的键和值都是用字符串对象。

**什么情况下会使他们发生转换**

- hash对象中保存的键值对的键和值的字符串长度都小于64
- hash对象中保存的键值对少于512个

满足以上两点键会使用ziplist，反之将会转化为hashtable，不过量值不是固定的，可以通过
配置文件进行修改，`hash-max-ziplist-value` and `hash-max-ziplist-entries`。

### 集合（Set）

**常用命令**

`OP key member [member ...]`

| op | 注释 |
|---|---|
|SADD|向集合中添加一个元素，member已经存在则会被忽略，key不存在将会被创建
|SISMEMBER|判断member是否为集合中的成员，是返回1其他情况返回0
|SPOP|随机移除一个元素
|SRANDMEMBER|只提供 key 参数时，返回随机一个元素；如果集合为空，返回nil，如果提供了count参数，那么返回一个数组；如果集合为空，返回空数组。
|SREM|移除一个或者多个元素
|SMOVE|原子性操作，移动原集合中的member到目标集合中，目标中存在这是单纯的将member移除。
|SCARD|返回集合中的数量
|SMEMBERS|返回结合中的所有成员
|SINTER|返回一个集合的全部成员，该集合是所有给定集合的交集。不存在的 key 被视为空集。
|SUNION|返回一个集合的全部成员，该集合是所有给定集合的并集。
|SDIFF|返回一个集合的全部成员，该集合是所有给定集合之间的差集。
|SDIFFSTORE|与SDIFF相识，但它将结果保存到 destination 集合，而不是简单地返回结果集。

**实现方式**

集合的底层实现是由intset和hashtable实现，使用整数集合时所有元素都被放在集合里面，
使用hashtable时，将会被保存在字典中，字典的key将会保存每一个元素，字典的value值将会被
置null。

**何时发生转换**

- 当集合中所有元素都是整数时
- 当集合中保存的数量超过512时

同时满足以上两点那么redis将会使用intset保存集合中的元素。同样这个是可配置的使用用
`set-max-intset-entries`进行配置。

### 有序集合（sort set）

**常用命令**

| op | 注释 |
|---|---|
|ZADD | 将member及其score放入到有序key的集合中，如果某个 member 已经是有序集的成员，那么更新这个 member 的 score 值，并通过重新插入这个 member 元素，来保证该 member 在正确的位置上。
|ZSCORE | 返回有序集 key 中，成员 member 的 score 值。
|ZCARD | 返回有序集 key 的基数。
|ZCOUNT | 返回有序集 key 中， score 值在 min 和 max 之间(默认包括 score 值等于 min 或 max )的成员的数量。
|ZRANGE | 返回有序集 key 中，指定区间内的成员。
|ZREVRANGE | 返回有序集 key 中，指定区间内的成员，逆序排列。
|ZRANGEBYSCORE | 返回有序集 key 中，指定区间内的成员。
|ZREVRANGEBYSCORE | 返回有序集 key 中，所有 score 值介于 min 和 max 之间(包括等于 min 或 max )的成员。有序集成员按 score 值逆序。
|ZRANK | 返回有序集 key 中成员 member 的排名。其中有序集成员按 score 值递增(从小到大)顺序排列。
|ZREVRANK | 返回有序集 key 中成员 member 的排名。其中有序集成员按 score 值逆序。
|ZREM | 移除有序集 key 中的一个或多个成员，不存在的成员将被忽略。key存在但不是有序集合时将会报错
|ZREMRANGEBYRANK | 移除有序集 key 中范围内的成员，不存在的成员将被忽略。按照rank的排序
|ZREMRANGEBYSCORE | 移除有序集 key 中的范围内的成员，不存在的成员将被忽略。按照score排序
|ZUNIONSTORE | 交集，并将结果存储到新的有序集合中

**实现方式**

有序集合内部由压缩列表、字典和[跳表][2]实现的。
- 在ziplist实现中，每个集合元素使用两个紧挨在一起的压缩列表实现，第一个节点保存元素的
成员，第二个保存元素的score，内部按照score的大小进行排列，score大的放在靠近表尾，小
的放在表头。
- 在skiplist的实现中，使用zset作为地层结构，每个zset包含了一个字典和一个跳表。在跳表中
节点的object属性保存了元素的成员，而跳表的score属性保存了有序集合元素的score。在字典中
每个字典的key将会保存元素，value将会用来保存score，这样就创建了一个元素到score的映射
，加快`zscore`的速度。虽然在redis的有序集合skiplist实现中同时使用了两种数据结构，
不过两种结构时对象项共享的，也就是锁元素的String对象和score的float对象都是被共享的，
所以不会产生内存浪费这种现象。

**为什么要同时使用两种数据结构实现有序集合？**

源码中的注释大概意思就是，为了效率。并且明确的指出了两种数据结构使用的是共享SDS，也就
是说redis在管理一个字符串时另一个也会被影响。至于为什么使用两种，可以这样理解，若单独使用
字典来实现，那以O(1)的时间获取指定元素的score将会被保持，但是`ZRANGE`使用这样的范围型
操作时，由于字典无序，那么也就是说每次获取前都要进行排序，至少需要O(log(n))。同样若单
独使用跳表实现，那么每次查找元素对应的score将会花费O(log(n))。所以为了让有序集合的查找
和范围操作快速执行，redis使用了两种数据结构。

**何时发生转换**

- 当有序集合中元素数量小于128
- 当有序集合中每个元素的长度小于64

同时满足以上两点那么redis将会使用ziplist保存集合中的元素。反之使用skiplist同样这个是可配置的使用用
`zset-max-ziplist-entries`和`zset-max-ziplist-value`进行配置。

[1]:http://zhangtielei.com/posts/blog-redis-quicklist.html
[2]:http://zhangtielei.com/posts/blog-redis-skiplist.html