# 笔记

## 转义问题

通常选择将用户数据序列化为JSON格式后存入redis中，并且JSON格式要求键值都用双引号进行括起来。

在redis-cli中查看键内容时，又以字符串的形式进行显示，当显示JSON数据时，会对JSON的双引号都进行转义。

为了方便查看，使用`redis-cli --raw`启动客户端时，不会在内容显示时增加双引号，因此JSON数据中的双引号也不需要`\`来进行转义了。

## 官网教程

- **所有由单个命令实现的Redis操作都具有原子性**，包括对更复杂数据结构的操作。所以当你使用Redis命令修改一些值，不必考虑并发问题。

### 基本操作

```bash
SET server:name "fido"
GET server:name # "fido"
EXISTS server:name # 1表示存在，0表示不存在
```

```bash
SET connections 10
INCR connections
DEL connections
INCRBY connections 100 # 增加100
DECR connections
DECRBY connections 10

SET connections 10
DEL connections # 1表示操作成功
GET connections # nil
INCR connections # 1表示值为1，因此我没必要先新建一个key并映射0，然后进行加1操作，可以直接进行加1操作，如果key不存在则新建，变成了单命令的原子操作
```

```bash
SET resource:lock "Redis Demo"
EXPIRE resource:lock 120 # 设置一个key的过期时间
TTL resource:lock # 返回直到被删除的秒数，-2表示不存在，-1表示永久存在
SET resource:lock "Redis Demo2" # 如果再次设置该键，则其过期时间需要重新设置

SET resource:lock "Redis Demo 3" EX 5 # 设置值的同时设置TTL
PERSIST resource:lock # 取消过期时间
```

### 列表

```bash
RPUSH friends "Alice" # 直接新建一个list并添加元素
LPUSH friends "Sam" # 添加元素的同时会返回当前list长度
RPUSH friends 1 2 3 # 添加多个元素

LRANGE friends 0 -1
LRANGE friends 0 1
LRANGE friends 1 2

LPOP friends # 删除并返回给客户端
RPOP friends
LLEN friends # 返回list长度

LTRIM mylist 0 99 # 修剪列表，保留0-99共100个元素，这样可以确保list当中元素不会超过100个
```

### 无序集合

```bash
SADD superpowers "flight" # 直接新建一个set并添加元素，1表示添加成功，0表示已存在无法添加
SADD superpowers "x-ray vision" "reflexes" # 同样可以指定多个元素，将返回添加成功的元素个数
SREM superpowers "reflexes" # 移除一个指定元素，1表示存在，0表示不存在
SMEMBERS superpowers
SISMEMBER superpowers "flight"
SUNION superpowers birdpowers # 返回并集
SPOP letters 2 # 随机移除2个元素，因为set是无序的
SRANDMEMBER letters 3 # 随机返回3个元素，如果参数为负数，则可能返回重复元素

SCARD myset # 返回set当中的个数
SINTER myset1 myset2 # 返回交集
```

### 有序集合

```bash
ZADD hackers 1912 "Alan Turing" # 新建一个有序set，每个元素都有一个关联score，根据这个score进行排序，score相同则作string比较
ZADD hackers 1957 "Sophie Wilson" # 如果已存在相同元素，则更新该元素的score
ZADD hackers 1916 "Claude Shannon"
ZRANGE hackers 0 -1
ZSCORE hackers "Alan Turing"
```

### 散列

```bash
HSET user:1000 name "John Smith" # 新建一个hash(user:1000)，并添加一个string field及其对应的string value
HGETALL user:1000
HMSET user:1001 name "Mary Jones" password "hidden" email "mjones@example.com" # 新建并添加多个string field和string value
HGET user:1001 name

HSET user:1000 visits 10 # 设置一个数值类型的字段值
HINCRBY user:1000 visits 1 # 对于数值类型的字段值存在自增操作
HINCRBY user:1000 visits 10
HDEL user:1000 visits
HINCRBY user:1000 visits 1
```

## 克隆一个twitter

在redis中没有table这一概念。

首先设计用户，它有用户ID、用户名、密码。其中，**使用用户唯一ID来命名包含用户数据的哈希key，这是键值存储中非常常见的设计模式**。

```bash
INCR next_user_id # 使用一个INCR命令保证全局用户ID的唯一
HMSET user:${上面命令的返回值} username cat password cat # 每有一个新用户便新建一个user:ID的map，存储用户名及密码
HSET users cat ${上上面命令的返回值} # 希望通过用户名获取用户ID，为此每有一个新用户，便向users这个map中放入用户名及对应ID
```

设计关注和被关注人，同时需要记录被关注时间和关注时间，为此使用sorted set最合适，使用时间作为score，用户ID作为元素。

同样使用包含用户唯一ID的key name来命名每个用户的关注人和被关注人的sorted set。

```bash
ZADD followers:1000 1401267618 1234
ZADD following:1000 1500000000 1234
```

设计每个用户的推文，需要按照时间顺序访问这些推文，从最新到最老，最适合的数据结构是list，借助LPUSH的将最新的推文ID放到其中，另外借助LRANGE可以实现分页功能。

```text
posts:1000 => a List of post ids - every new post is LPUSHed here.
```

设计用户认证，我们将用户cookie作为用户数据的一部分，在创建用户时使用一个随机、不可猜测的字符串作为auth的value，并且需要能够使用用户cookie来找到id的数据结构。

为了验证用户身份，可以通过如下步骤：

- 通过表单获取用户名和密码。
- 验证username字段是否存在于users中。
- 如果存在，我们可以获得用户ID。
- 检查user:ID中password字段是否匹配，不匹配则返回错误消息。
- 验证成功，取出user:ID中auth字段的值作为用户的cookie。

```bash
HSET user:1000 auth fea5e81ac8ca77622bed1c2132a021f9
HSET auths fea5e81ac8ca77622bed1c2132a021f9 1000
```

## 数据类型

### 字符串

字符串值可以用于缓存HTML页面和图片，但大小不能超过512MB。

```bash
SET mykey newval nx # 如果键存在则赋值失败
SET mykey newval nx # 即使键存在赋值仍成功

SET counter 100
INCR counter # 将字符串值解析为整型，对其加一，将最后获得值设为新值，其它增减操作同理

GETSET user:counter 0 # GETSET在设置新值时同时获取旧值，搭配INCR可以实现记录网站每小时的访客次数
INCR user:counter
INCR user:counter
GETSET user:counter 0 # 2

# 批量设置和批量获取可以有效减少延迟
MSET a 10 b 20 c 30
MGET a b c
```

### 修改和查询键空间

```bash
exists mykey
del mykey
type mykey # 查询值类型
```

### 键过期

- They can be set both using seconds or milliseconds precision.
- However the expire time resolution is always 1 millisecond.
- Information about expires are replicated and persisted on disk, the time virtually passes when your Redis server remains stopped (this means that Redis saves the date at which a key will expire).

上述第三点的意思是，即使 Redis 服务器停止工作，其之后的时间仍然会被计算在过期时间内。

```bash
expire key 5 # expire指令可以给已有过期时间的键设置一个新的过期时间
```

### 列表

Redis 列表底层存储结构是链表，**理由是对于一个数据库系统而言，至关重要的就是能够以非常快的方式将元素添加到很长的列表中**。缺点就是那个缺点，按索引访问时慢。

Redis 列表的一个强大优势是能够在恒定时间内获取恒定长度的列表。

当需要访问大量元素的中间部分很重要时，可以使用 Sorted sets。

```bash
rpush mylist 1 2 3 4 5 "foo bar" # push操作支持可变参数
```

### 列表的常见用例

以下是两个非常有代表性的用例：

- 记住用户发布到社交网络上的最新更新。
- 生产者和消费者模式。Redis 具有特殊的列表命令，以使此用例更可靠和高效。

为了逐步描述一个常见的用例，假设您的主页显示了在照片共享社交网络中发布的最新照片，并且您想加快访问速度。

- 每次用户发布新照片时，我们使用`LPUSH`将其ID添加到列表中。
- 当用户访问主页时，我们使用`LRANGE 0 9`获取最新发布的10个项目。

### 限制列表

```bash
rpush mylist 1 2 3 4 5
ltrim mylist 0 2 # 仅保留范围内的元素并丢弃其它元素
lrange mylist 0 -1
```

一个简单而有用的方式就是使用`lpush`和`ltrim`添加新元素并丢弃超出限制的元素。之后使用`lrange`访问最新的元素，而无需记住非常旧的数据。

> Note: while LRANGE is technically an O(N) command, accessing small ranges towards the head or the tail of the list is a constant time operation.

### 列表的阻塞操作

仅用`lpush`和`rpop`就可以实现生产者-消费者模式，但是`rpop`命令缺点就是在列表为空时需要轮询重试。

Redis 提供了阻塞版本的`pop`命令，`brpop`和`blpop`，这两个命令在列表为空时将阻塞，或者在超过指定时间后返回空。

```bash
brpop tasks 5 # 如果5秒后没有可用元素，则返回
```

上述指令在`timeout`参数为零时将永久等待元素到来，而且该指令可以指定多个列表同时等待，并在其中一个列表收到第一个元素时返回（按照参数顺序）。

`brpop`会返回两个值，第一个值是列表名，第二个值是元素，因为该命令可以同时等待多个列表，需要有一个返回值进行表述。

`lmove`于`6.2.0`版本提供，用于构建更安全的队列和旋转队列，`blmove`是它的阻塞版本。

### 自动创建和删除键

适用于多元素组成的数据类型——Lists, Streams, Sets, Sorted Sets and Hashes.

1. 当我们将元素添加到聚合数据类型时，如果目标键不存在，则在添加元素之前会创建一个空的聚合数据类型。
2. 当我们从聚合数据类型中删除元素时，如果该值保持为空，则键将自动销毁。流数据类型是此规则的唯一例外。

    ```bash
    lpush mylist 1 2 3
    exists mylist # 1
    lpop mylist
    lpop mylist
    lpop mylist
    exists mylist # 0
    ```

3. 调用只读命令，如`llen`(它返回列表的长度)，或使用空键删除元素的写命令，都会产生相同的结果，就好像该键持有命令期望找到的空聚合类型一样。

    ```bash
    del mylist
    llen mylist # 0
    lpop mylist # nil
    ```

### 哈希映射

```bash
hincrby user:1000 birthyear 10 # 哈希映射字段同样支持自增操作
```

> It is worth noting that small hashes (i.e., a few elements with small values) are encoded in special way in memory that make them very memory efficient.

### 集合

集合是字符串的无序集合。**集合非常重要的特性就是可以计算交集、并集、差集**。

集合很适合表达对象之间的关系。例如，我们可以很容易地使用集合来实现标记。比如为我们想要标记的每个对象设置一个集合。该集合包含与对象相关联的标记的ID。

其中一个例子是给新闻文章加标签。假设我们有一个标签ID映射到标签名称的哈希映射。

```bash
sadd news:1000:tags 1 2 5 77 # 第1000号新闻的标签是1、2、5、77
sadd tags:1:news 1000 # 反向关联标签和新闻
sadd tags:2:news 1000
sadd tags:5:news 1000
sadd tags:77:news 1000
sinter tags:1:news tags:2:news tags:5:news tags:77:news # 使用交集寻找包含参数标签的新闻
```

还有一个例子是扑克牌游戏。

```bash
sadd deck C1 C2 C3 C4 C5 C6 C7 C8 C9 C10 CJ CQ CK ...
spop deck 5 # 从52张扑克牌当中随机弹出5张牌给玩家
# 直接弹出则在下一局游戏开始时需要重新填充集合，一种更好的做法是使用sunionstore，
# 它可以执行多个集合之间的并集并将结果存储到目标集合当中，由于此处只有单个集合，
# 因此实现了复制牌组到第一局牌组的效果。
sunionstore game:1:deck deck
scard game:1:deck # 查看集合数量
srandmember game:1:deck 5 # 实现随机返回5个非重复元素，参数为负数将随机返回5个可能重复的元素
```

### 排序集

> Sorted sets are a data type which is similar to a mix between a Set and a Hash. Like sets, sorted sets are composed of unique, non-repeating string elements, so in some sense a sorted set is a set as well.  
> However while elements inside sets are not ordered, every element in a sorted set is associated with a floating point value, called the score (this is why the type is also similar to a hash, since every element is mapped to a value).

排序规则如下：

- 如果A和B具有两个不同的score，则谁的score大谁排在后面。
- 如果score相同，则考虑两个字符串在字典序上的排序，因为不会存在相同元素，因此肯定可以进行排序。

> Implementation note: Sorted sets are implemented via a dual-ported data structure containing both a skip list and a hash table, so every time we add an element Redis performs an O(log(N)) operation.

```bash
zrange hackers 0 -1 # 正序输出
zrange hackers 0 -1 withscores # 顺带输出score
zrevrange hackers 0 -1 # 倒序输出
```

### 排序集的范围操作

```bash
zrangebyscore hackers -inf 1950 # 输出所有出生年份在1950（包含）以内的个人
zramrangebyscore hackers 1940 1960 # 删除范围之内的元素并返回删除数量
zrank hackers "Anita Borg" # 输出其在有序集合当中的位置
zrevrank hackers "Anita Borg" # 反向输出其在有序集合当中的位置
```

### 排序集的字典序

在 Redis 的2.8版本引入了一个新功能：按照字典序去获取排序集的范围元素。

字典序的内部实现方式是通过C语言当中`memcmp`函数。

```bash
zadd hackers 0 "Alan Kay" 0 "Sophie Wilson" 0 "Richard Stallman" 0 # 分数相同，将按照字典序进行排序
zrangebylex hackers [B [P # 获取开头从B到P的黑客名字，注意[是闭区间的意思，相应存在(表示开区间
```

这个特性很重要，因为它允许我们使用排序集作为泛型索引。例如，如果你想用一个128位的无符号整数参数索引元素，你所需要做的就是用相同的分数(例如0)将元素添加到一个排序的集合中，但是有一个16字节的前缀，由128位的大端数组成。由于大端中的数字在按字典顺序排序时(按原始字节顺序)实际上也是按数字顺序排序的，所以您可以请求128位空间中的范围，并获得丢弃前缀的元素值。

### 排序集更新分数：排行榜

排序集的score可以随时更新。仅对已经包含在排序集中的元素调用ZADD，就会以`O(log(N))`的时间复杂度更新其score(和位置)。因此，排序集适用于有大量更新的情况。

由于这个特点，一个常见的用例就是排行榜。典型的应用是一款Facebook游戏，你结合了根据高分对用户进行排序的功能，再加上`zrank`操作，以便显示排名前n的用户和用户在排行榜上的排名(例如，“你在这里获得了第4932位的最佳分数”)。

### 位图

位图不是实际的数据类型，而是String类型上定义的一组面向位的操作。

> Since strings are binary safe blobs and their maximum length is 512 MB, they are suitable to set up to 2^32 different bits.

位图的优点之一是，在存储信息时通常可以节省大量的空间。例如：在以增量用户ID表示不同用户的系统中，仅使用512MB内存就可以记住40亿用户的信息。

```bash
setbit key 10 1 # 设置第10位为1，如果地址位超出当前字符串长度，该命令会自动扩大字符串
getbit key 10 # 1
getbit key 11 # 0
getbit key 0 # 0
```

1. `bitop`在不同字符串之间执行位操作，`AND`、`OR`、`XOR`、`NOT`。
2. `bitcount`计算位为1的数量。
3. `bitops`找到位为1或0的第一个位置。

### HyperLogLog

用于快速统计非重复元素的一种概率数据结构。

```bash
pfadd hll a b c d a b c d e
pfcount 5
```

常见用例是计算用户每天在搜索表单当中执行的唯一查询。
