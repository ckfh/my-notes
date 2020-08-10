# 笔记

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
```

```bash
SADD superpowers "flight" # 直接新建一个set并添加元素，1表示添加成功，0表示已存在无法添加
SADD superpowers "x-ray vision" "reflexes" # 同样可以指定多个元素
SREM superpowers "reflexes" # 移除一个指定元素，1表示存在，0表示不存在
SMEMBERS superpowers
SISMEMBER superpowers "flight"
SUNION superpowers birdpowers # 合并两个set
SPOP letters 2 # 随机移除2个元素，因为set是无序的
SRANDMEMBER letters 3 # 随机返回3个元素，如果参数为负数，则可能返回重复元素
```

```bash
ZADD hackers 1912 "Alan Turing" # 新建一个有序set，每个元素都有一个关联score，根据这个score进行排序
ZADD hackers 1957 "Sophie Wilson"
ZADD hackers 1916 "Claude Shannon"
ZRANGE hackers 0 -1
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
