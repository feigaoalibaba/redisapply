# redis

#单机数据库的实现


#expire key redis 主键失效原理及实现机制
##失效时间的控制
Redis的key 失效时间会受到命令的影响吗？当然了

    - 首先，在通过 DEL 命令删除一个主键时，失效时间自然会被撤销
    - 其次，在一个设置了失效时间的主键被更新覆盖时，该主键的失效时间也会被撤销。但需要注意的是，这里所说的是主键被更新覆盖，而不是主键对应的 Value 被更新覆盖，因此 SET、MSET 或者是 GETSET 可能会导致主键被更新覆盖，而像 INCR、DECR、LPUSH、HSET 等都是更新主键对应的值，这类操作是不会触碰主键的失效时间的。
    - 此外，还有一个特殊的命令就是 RENAME，当我们使用 RENAME 对一个主键进行重命名后，之前关联的失效时间会自动传递给新的主键，但是如果一个主键是被RENAME所覆盖的话（如主键 hello 可能会被命令 RENAME world hello 所覆盖），这时被覆盖主键的失效时间会被自动撤销，而新的主键则继续保持原来主键的特性。

##失效的内部实现
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
### passive way 消极方法
- 在passive way 中， redis在实现GET、MGET、HGET、LRANGE等所有涉及到读取数据的命令时都会调用 expireIfNeeded，它存在的意义就是在读取数据之前先检查一下它有没有失效，如果失效了就删除它。
- expireIfNeeded函数中调用的另外一个函数propagateExpire，这个函数用来在正式删除失效主键之前广播这个主键已经失效的信息，这个信息会传播到两个目的地：
1. 一个是发送到AOF文件，将删除失效主键的这一操作以DEL Key的标准命令格式记录下来；
2. 另一个就是发送到当前Redis服务器的所有Slave，同样将删除失效主键的这一操作以DEL Key的标准命令格式告知这些Slave删除各自的失效主键。从中我们可以知道，所有作为Slave来运行的Redis服务器并不需要通过消极方法来删除失效主键，它们只需要对Master唯命是从就OK了！

代码段二：

	int expireIfNeeded(redisDb *db, robj *key) {
		获取主键的失效时间
		long long when = getExpire(db,key);
		假如失效时间为负数，说明该主键未设置失效时间（失效时间默认为-1），直接返回0
		if (when < 0) return 0;
		假如Redis服务器正在从RDB文件中加载数据，暂时不进行失效主键的删除，直接返回0
		if (server.loading) return 0;
		假如当前的Redis服务器是作为Slave运行的，那么不进行失效主键的删除，因为Slave
		上失效主键的删除是由Master来控制的，但是这里会将主键的失效时间与当前时间进行
		一下对比，以告知调用者指定的主键是否已经失效了
		if (server.masterhost != NULL) {
			return mstime() > when;
		}
		如果以上条件都不满足，就将主键的失效时间与当前时间进行对比，如果发现指定的主键
		还未失效就直接返回0
		if (mstime() <= when) return 0;
		如果发现主键确实已经失效了，那么首先更新关于失效主键的统计个数，然后将该主键失
		效的信息进行广播，最后将该主键从数据库中删除
		server.stat_expiredkeys++;
		propagateExpire(db,key);
		return dbDelete(db,key);
	}

	void propagateExpire(redisDb *db, robj *key) {
		robj *argv[2];
		shared.del是在Redis服务器启动之初就已经初始化好的一个常用Redis对象，即DEL命令
		argv[0] = shared.del;
		argv[1] = key;
		incrRefCount(argv[0]);
		incrRefCount(argv[1]);
		检查Redis服务器是否开启了AOF，如果开启了就为失效主键记录一条DEL日志
		if (server.aof_state != REDIS_AOF_OFF)
			feedAppendOnlyFile(server.delCommand,db->id,argv,2);
		检查Redis服务器是否拥有Slave，如果是就向所有Slave发送DEL失效主键的命令，这就是
		上面expireIfNeeded函数中发现自己是Slave时无需主动删除失效主键的原因了，因为它
		只需听从Master发送过来的命令就OK了
		if (listLength(server.slaves))
			replicationFeedSlaves(server.slaves,db->id,argv,2);
		decrRefCount(argv[0]);
		decrRefCount(argv[1]);
	}

### Active Way
消极方法的缺点是，如果key 迟迟不被访问，就会占用很多内存空间. 所以就产生的积极的方式(Active Way)：此方法利用了redis的时间事件，即每隔一段时间就中断一下完成一些指定操作，其中就包括检查并删除失效主键。

* 时间事件

创建时间事件, 回调函数就是serverCron，它在Redis服务器启动时创建，每秒的执行次数由宏定义REDIS_DEFAULT_HZ来指定，默认每秒钟执行10次。


* 使用activeExpireCycle 清除失效key

其实现原理是从Redis中每个数据库的expires字典表中，随机抽样REDIS_EXPIRELOOKUPS_PER_CRON（默认值为10）个设置了失效时间的主键，检查它们是否已经失效并删除掉失效的主键，如果失效主键个数占本次抽样个数的比例超过25%，它会继续进行下一轮的随机抽样和删除，直到刚才的比例低于25%才停止对当前数据库的处理，转向下一个数据库。

> 注意，activeExpireCycle函数不会试图一次性处理Redis中的所有数据库，而是最多只处理REDIS_DBCRON_DBS_PER_CALL（默认值为16），此外activeExpireCycle函数还有处理时间上的限制，不是想执行多久就执行多久，凡此种种都只有一个目的，那就是避免失效主键删除占用过多的CPU资源。
### Memcached删除失效主键的方法与Redis有何异同？
首先，Memcached在删除失效主键时采用的是消极方法
其次，Memcached与Redis在主键失效机制上的最大不同是，Memcached不会像Redis那样真正地去删除失效的主键，而只是简单地将失效主键占用的空间回收。直到有新数据要写入时，才复用失效空间。
LRU内存回收: 如果空间用光了，Memcached还可以通过LRU机制来回收那些长期得不到访问的空间。

Redis在出现OOM时同样可以通过配置maxmemory-policy这个参数来决定是否采用LRU机制来回收内存空间(放到硬盘)

### LRU算法
LRU是Least Recently Used最近最少使用算法。源于操作系统使用的一种算法，对于在内存中但最近又不用的数据块叫做LRU，操作系统会将那些属于LRU的数据移出内存，从而腾出空间来加载另外的数据。
算法：对key 按失效时间排序，然后取最先失效的key
还有一种FIFO（先入先出）算法: 只适用expire_time = current_time + valid_time(固定) 的情况，这样就可以避免对key 进行expire_time 排序了（它本来就是有序的）

#RDB 持久化和 AOF 持久化的实现原理
 
