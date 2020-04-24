# hash散列对象

## 简介

---
源码位置：t_hash.c/server.h

|命令|功能|时间复杂度|
|---|---|---|
|HSET|设置key指定的哈希集中指定字段的值|O(1)|
|HSETNX|只在key指定的哈希集中不存在指定的字段时，设置字段的值|O(1)|
|HMSET|设置key指定的哈希集中指定多个字段的值|O(N)，N为字段数|
|HGET|返回key指定的哈希集中该字段所关联的值|O(1)|
|HMGET|返回key指定的哈希集中指定多个字段的值|O(N)，N为字段数|
|HGETALL|返回key指定的哈希集中所有的字段和值|O(N)，N为hash的size|
|HVALS|返回key指定的哈希集中所有字段的值|O(N)，N为hash的size|
|HDEL|从key指定的哈希集中移除指定的域|O(N)，N是被删除的字段数量|
|HEXISTS|返回hash里面field是否存在|O(1)|
|HKEYS|返回key指定的哈希集中所有字段的名字|O(N)，N为hash的size|
|HLEN|返回key指定的哈希集包含的字段的数量|O(1)|
|HSCAN|用于迭代Hash类型中的键值对|O(1)|
|HSTRLEN|返回hash指定field的value的字符串长度，如果hash或者field不存在，返回0|O(1)|
|HINCRBY|增加key指定的哈希集中指定字段的数值|O(1)|
|HINCRBYFLOAT|为指定key的hash的field字段值执行float类型的increment加|O(1)|

</br>
</br>

## 结构体与宏定义

---

``` c
void hsetCommand(client *c); // hset命令
void hsetnxCommand(client *c); // hsetnx命令
void hmsetCommand(client *c); // hmset命令
void hgetCommand(client *c); // hget命令
void hmgetCommand(client *c); // hmget命令
void hgetallCommand(client *c); // hgetall命令
void hvalsCommand(client *c); // hvals命令
void hdelCommand(client *c); // hdel命令
void hexistsCommand(client *c); // hexists命令
void hkeysCommand(client *c); // hkeys命令
void hlenCommand(client *c); // hlen命令
void hscanCommand(client *c); // hscan命令
void hstrlenCommand(client *c); // hstrlen命令
void hincrbyCommand(client *c); // hincrby命令
void hincrbyfloatCommand(client *c); // hincrbyfloat命令
```
