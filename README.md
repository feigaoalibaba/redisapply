# redis

#单机数据库的实现

#expire key redis 主键失效原理及实现机制

#失效时间的控制
Redis的key 失效时间会受到命令的影响吗？当然了

    - 首先，在通过 DEL 命令删除一个主键时，失效时间自然会被撤销
    - 其次，在一个设置了失效时间的主键被更新覆盖时，该主键的失效时间也会被撤销。但需要注意的是，这里所说的是主键被更新覆盖，而不是主键对应的 Value 被更新覆盖，因此 SET、MSET 或者是 GETSET 可能会导致主键被更新覆盖，而像 INCR、DECR、LPUSH、HSET 等都是更新主键对应的值，这类操作是不会触碰主键的失效时间的。
    - 此外，还有一个特殊的命令就是 RENAME，当我们使用 RENAME 对一个主键进行重命名后，之前关联的失效时间会自动传递给新的主键，但是如果一个主键是被RENAME所覆盖的话（如主键 hello 可能会被命令 RENAME world hello 所覆盖），这时被覆盖主键的失效时间会被自动撤销，而新的主键则继续保持原来主键的特性。

#失效的内部实现
 Redis擅长失效主键的方法主要有两种：
 消极方法（passive way），在主键被访问时如果发现它已经失效，那么就删除它
 积极方法（active way），周期性地从设置了失效时间的主键中选择一部分失效的主键删除
 主动删除：当前已用内存超过maxmemory限定时，触发主动清理策略，该策略由启动参数的配置决定
 
 主键具体的失效时间全部都维护在expires这个字典表中。
 
 typedef struct redisDb {
 		dict *dict; //key-value
 		dict *expires;  //维护过期key
 		dict *blocking_keys;
 		dict *ready_keys;
 		dict *watched_keys;
 		int id;
 	} redisDb;
 
RDB 持久化和 AOF 持久化的实现原理
