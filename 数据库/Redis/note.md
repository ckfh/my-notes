# 笔记

## 转义问题

通常选择将用户数据序列化为JSON格式后存入redis中，并且JSON格式要求键值都用双引号进行括起来。

在redis-cli中查看键内容时，又以字符串的形式进行显示，当显示JSON数据时，会对JSON的双引号都进行转义。

为了方便查看，使用`redis-cli --raw`启动客户端时，不会在内容显示时增加双引号，因此JSON数据中的双引号也不需要`\`来进行转义了。

## 官网教程

- 所有由单个命令实现的Redis操作都具有原子性，包括对更复杂数据结构的操作。所以当你使用Redis命令修改一些值，不必考虑并发问题。

```bash
SET server:name "fido"
GET server:name
EXISTS server:name # 1表示存在，0表示不存在
```

```bash
SET connections 10
INCR connections
DEL connections
INCRBY connections 100
DECR connections
DECRBY connections 10

SET connections 10
DEL connections # 1表示操作成功
GET connections # nil
INCR connections # 1表示值为1，因此我没必要先新建一个key为0，然后进行加1操作，可以直接进行加1操作，如果key不存在则新建，变成了单命令的原子操作
```

```bash
SET resource:lock "Redis Demo"
EXPIRE resource:lock 120 # 设置一个key的过期时间
TTL resource:lock # 返回直到被删除的秒数，-2表示不存在，-1表示永久存在

SET resource:lock "Redis Demo 3" EX 5 # 设置值的同时设置TTL
PERSIST resource:lock # 取消过期时间
```

```bash
RPUSH friends "Alice" # 直接新建一个list并添加元素
LPUSH friends "Sam"
RPUSH friends 1 2 3 # 指定多个元素

LRANGE friends 0 -1
LRANGE friends 0 1
LRANGE friends 1 2

LPOP friends # 删除并返回给客户端
RPOP friends
LLEN friends

LPUSH mylist someelement # 从开头添加一个元素
LTRIM mylist 0 99 # 修剪列表，保留0-99共100个元素，这样可以确保list当中元素不会超过100个
```

```bash
SADD superpowers "flight" # 直接新建一个set并添加元素，1表示添加成功，0表示已存在无法添加
SADD superpowers "x-ray vision" "reflexes" # 同样可以指定多个元素
SREM superpowers "reflexes" # 移除一个指定元素，1表示存在，0表示不存在
SMEMBERS superpowers
SISMEMBER superpowers "flight"
SUNION superpowers birdpowers # 返回并集
SPOP letters 2 # 随机移除2个元素，因为set是无序的
SRANDMEMBER letters 3 # 随机返回3个元素，如果参数为负数，则可能返回重复元素

SCARD myset # 返回set当中的个数
SINTER myset1 myset2 # 返回交集
```

```bash
ZADD hackers 1912 "Alan Turing" # 新建一个有序set，每个元素都有一个关联score，根据这个score进行排序，score相同则作string比较
ZADD hackers 1957 "Sophie Wilson" # 不能添加重复元素，但是能更新score
ZADD hackers 1916 "Claude Shannon"
ZRANGE hackers 0 -1
ZSCORE hackers "Alan Turing"
```

```bash
HSET user:1000 name "John Smith" # 新建一个map(user:1000)，并添加一个string field及其对应的string value
HGETALL user:1000
HMSET user:1001 name "Mary Jones" password "hidden" email "mjones@example.com" # 新建并添加多个field和value
HGET user:1001 name

HSET user:1000 visits 10
HINCRBY user:1000 visits 1
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
