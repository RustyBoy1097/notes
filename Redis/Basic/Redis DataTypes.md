# Redis 数据类型
Redis支持五种数据类型：string（字符串）、hash（哈希）、list（列表）、set（集合）及zset(sorted set：有序集合）

## String（字符串）

* string是 redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value。
* string类型是二进制安全的。意思是 redis 的 string 可以包含任何数据。比如jpg图片或者序列化的对象。
* string类型是 Redis 最基本的数据类型，string 类型的值最大能存储512MB。

```
redis 127.0.0.1:6379> SET test_key "test_value"
OK
redis 127.0.0.1:6379> GET test_key
"test_value"
```

## Hash（哈希）

* Redis hash 是一个键值(key=>value)对集合。
* Redis hash 是一个 string 类型的 field 和 value 的映射表，hash 特别适合用于存储对象。

实例
DEL runoob 用于删除前面测试用过的 key，不然会报错：(error) WRONGTYPE Operation against a key holding the wrong kind of value


```
redis 127.0.0.1:6379> DEL test_key
redis 127.0.0.1:6379> HMSET test_key field1 "Hello" field2 "World"
"OK"
redis 127.0.0.1:6379> HGET test_key field1
"Hello"
redis 127.0.0.1:6379> HGET test_key field2
"World"
```

实例中我们使用了 Redis `HMSET`, `HGET` 命令，`HMSET` 设置了两个 `field=>value` 对, `HGET` 获取对应 `field` 对应的 `value`。

每个hash可以存储 2^32 -1 键值对（40多亿）。

## List（列表）

Redis 列表是简单的字符串列表，按照插入顺序排序。你可以添加一个元素到列表的头部（左边）或者尾部（右边）。

```
redis 127.0.0.1:6379> DEL test_key
redis 127.0.0.1:6379> lpush test_key redis
(integer) 1
redis 127.0.0.1:6379> lpush test_key mongodb
(integer) 2
redis 127.0.0.1:6379> lpush test_key rabbitmq
(integer) 3
redis 127.0.0.1:6379> lrange test_key 0 10
1) "rabbitmq"
2) "mongodb"
3) "redis"
```

列表最多可存储2^32 - 1 元素 (4294967295, 每个列表可存储40多亿)。

## Set（集合）

* Redis 的 Set 是 string 类型的无序集合。
* 集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)。

`sadd key member`

```
redis 127.0.0.1:6379> DEL test_key
redis 127.0.0.1:6379> sadd test_key redis
(integer) 1
redis 127.0.0.1:6379> sadd test_key mongodb
(integer) 1
redis 127.0.0.1:6379> sadd test_key rabbitmq
(integer) 1
redis 127.0.0.1:6379> sadd test_key rabbitmq
(integer) 0
redis 127.0.0.1:6379> smembers test_key

1) "redis"
2) "rabbitmq"
3) "mongodb"
```

注意：以上实例中 rabbitmq 添加了两次，但根据集合内元素的唯一性，第二次插入的元素将被忽略。

集合中最大的成员数为2^32 - 1(4294967295, 每个集合可存储40多亿个成员)。

## zset(sorted set：有序集合)

* Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。
* 不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。
* zset的成员是唯一的,但分数(score)却可以重复。

`zadd key score member `

```
redis 127.0.0.1:6379> DEL test_key
redis 127.0.0.1:6379> zadd test_key 0 redis
(integer) 1
redis 127.0.0.1:6379> zadd test_key 0 mongodb
(integer) 1
redis 127.0.0.1:6379> zadd test_key 0 rabbitmq
(integer) 1
redis 127.0.0.1:6379> zadd test_key 0 rabbitmq
(integer) 0
redis 127.0.0.1:6379> ZRANGEBYSCORE test_key 0 1000
1) "mongodb"
2) "rabbitmq"
3) "redis"
```

