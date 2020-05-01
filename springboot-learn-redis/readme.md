### # Redis
* SpringAOP+Redis实现分布式锁
* 监听Redis中key的过期事件
* 使用Redis生成流水号



* 缓存
    * 缓存穿透->查询不存在的值
        * 缓存不存在的值在cache中，下次直接从缓存中返回null
        * 在cache前面再加上一层bloomFilter，先从bloomFilter中查询key是否存在，存在则继续往下走cache，走DB。否则直接返回null。
    * 缓存击穿->对于访问量很大的热点数据，在某一时刻缓存失效，那么会有大量的请求打到DB，导致DB宕机
        * 在第一个查询请求上加上互斥锁，其他线程等第一个请求缓存完成之后再执行（如果查询结果为空，则获取互斥锁，查询完成之后将数据放入缓存，再讲互斥锁释放）
        * ~~或者使用消息队列~~
        * 后台维护一个定时任务来检查缓存的有效期，如果发现有缓存即将过期，则在过期之前更新缓存（缺点是会增加系统的复杂度）
    * 缓存雪崩->缓存服务器宕机或者某一时刻大量的热点数据失效
        * 做好集群缓存，提高缓存的可用性
        * 服务降级（对所有的问题都有效）
        * 对缓存的数据设置不同的失效时间（具体情况具体分析）

参考
* [https://blog.csdn.net/u010416101/article/details/79690404](https://blog.csdn.net/u010416101/article/details/79690404)
* [https://blog.csdn.net/liubenlong007/article/details/53690103](https://blog.csdn.net/liubenlong007/article/details/53690103)
* [https://www.cnblogs.com/xingzc/p/5988080.html](https://www.cnblogs.com/xingzc/p/5988080.html)



#### # Redis有哪些数据结构？
[字符串（strings）](http://www.redis.cn/topics/data-types-intro.html#strings)， [散列（hashes）](http://www.redis.cn/topics/data-types-intro.html#hashes)， [列表（lists）](http://www.redis.cn/topics/data-types-intro.html#lists)， [集合（sets）](http://www.redis.cn/topics/data-types-intro.html#sets)， [有序集合（sorted sets）](http://www.redis.cn/topics/data-types-intro.html#sorted-sets) 与范围查询， [bitmaps](http://www.redis.cn/topics/data-types-intro.html#bitmaps)， [hyperloglogs](http://www.redis.cn/topics/data-types-intro.html#hyperloglogs) 和 [地理空间（geospatial）](http://www.redis.cn/commands/geoadd.html) 索引半径查询。

### # 使用过Redis分布式锁么，它是什么回事？
先拿setnx（SET if Not exists）来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放。
~~更好的解决方案是`SETNX lock.foo <current Unix time + lock timeout + 1>`，将lock.foo的值设置为过期时间，则其他线程可以通过这个参数来判断该锁是否已经失效，如果发现已经超时，则其他线程可以将其删除。~~(没搞明白)
[http://www.redis.cn/commands/setnx.html](http://www.redis.cn/commands/setnx.html)

### # 假如Redis里面有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如果将它们全部找出来？
使用keys指令可以扫出指定模式的key列表。但是Redis的单线程的。keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用scan指令，scan指令可以无阻塞的提取出指定模式的key列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用keys指令长。(我理解的scan是类似于分页查询)

### # 使用过Redis做异步队列么，你是怎么用的？
一般使用list结构作为队列，rpush生产消息，lpop消费消息。当lpop没有消息的时候，要适当sleep一会再重试。
如果对方追问可不可以不用sleep呢？list还有个指令叫blpop，在没有消息的时候，它会阻塞住直到消息到来。 

### # 淘汰策略
在 redis 中，允许用户设置最大使用内存大小通过配置redis.conf中的maxmemory这个值来开启内存淘汰功能，在内存限定的情况下是很有用的。
设置最大内存大小可以保证redis对外提供稳健服务。
redis 内存数据集大小上升到一定大小的时候，就会施行数据淘汰策略。redis 提供 6种数据淘汰策略通过maxmemory-policy设置策略：
* volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
* volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
* volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
* allkeys-lru：从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰
* allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰
* no-enviction（驱逐）：禁止驱逐数据
redis 确定驱逐某个键值对后，会删除这个数据并将这个数据变更消息发布到本地（AOF 持久化）和从机（主从连接）

* [🔐 Redis实现分布式锁 ](http://www.redis.cn/topics/distlock.html)
* [📚 RDB与AOF](https://www.cnblogs.com/xingzc/p/5988080.html)

* TTL (Time To Live)



### # Redis提供了RDB持久化和AOF持久化
* RDB机制的优势和略施
RDB持久化是指在指定的时间间隔内将内存中的数据集快照写入磁盘。也是默认的持久化方式，这种方式是就是将内存中数据以快照的方式写入到二进制文件中,默认的文件名为dump.rdb。
可以通过配置设置自动做快照持久化的方式。我们可以配置redis在n秒内如果超过m个key被修改就自动做快照，下面是默认的快照保存配置
```
save 900 1     #900秒内如果超过1个key被修改，则发起快照保存
save 300 10    #300秒内容如超过10个key被修改，则发起快照保存
save 60 10000
```
#####  RDB文件保存过程
**redis调用fork,现在有了子进程和父进程。
父进程继续处理client请求，子进程负责将内存内容写入到临时文件。由于os的写时复制机制（copy on write)父子进程会共享相同的物理页面，当父进程处理写请求时os会为父进程要修改的页面创建副本，而不是写共享的页面。所以子进程的地址空间内的数 据是fork时刻整个数据库的一个快照。
当子进程将快照写入临时文件完毕后，用临时文件替换原来的快照文件，然后子进程退出。
client 也可以使用save或者bgsave命令通知redis做一次快照持久化。save操作是在主线程中保存快照的，由于redis是用一个主线程来处理所有 client的请求，这种方式会阻塞所有client请求。所以不推荐使用。**

另一点需要注意的是，每次快照持久化都是将内存数据完整写入到磁盘一次，并不 是增量的只同步脏数据。如果数据量大的话，而且写操作比较多，必然会引起大量的磁盘io操作，可能会严重影响性能。
* **优势**
  *  一旦采用该方式，那么你的整个Redis数据库将只包含一个文件，这样非常方便进行备份。比如你可能打算没1天归档一些数据。
  *  方便备份，我们可以很容易的将一个一个RDB文件移动到其他的存储介质上
  *  RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。
  *  RDB 可以最大化 Redis 的性能：父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作。
* **劣势**
  *  如果你需要尽量避免在服务器故障时丢失数据，那么 RDB 不适合你。 虽然 Redis 允许你设置不同的保存点（save point）来控制保存 RDB 文件的频率， 但是， 因为RDB 文件需要保存整个数据集的状态， 所以它并不是一个轻松的操作。 因此你可能会至少 5 分钟才保存一次 RDB 文件。 在这种情况下， 一旦发生故障停机， 你就可能会丢失好几分钟的数据。
  *  每次保存 RDB 的时候，Redis 都要 fork() 出一个子进程，并由子进程来进行实际的持久化工作。 在数据集比较庞大时， fork() 可能会非常耗时，造成服务器在某某毫秒内停止处理客户端； 如果数据集非常巨大，并且 CPU 时间非常紧张的话，那么这种停止时间甚至可能会长达整整一秒。 虽然 AOF 重写也需要进行 fork() ，但无论 AOF 重写的执行间隔有多长，数据的耐久性都不会有任何损失。

#####  AOF文件保存过程
redis会将每一个收到的写命令都通过write函数追加到文件中(默认是 appendonly.aof)。
当redis重启时会通过重新执行文件中保存的写命令来在内存中重建整个数据库的内容。当然由于os会在内核中缓存 write做的修改，所以可能不是立即写到磁盘上。这样aof方式的持久化也还是有可能会丢失部分修改。不过我们可以通过配置文件告诉redis我们想要 通过fsync函数强制os写入到磁盘的时机。有三种方式如下（默认是：每秒fsync一次）
```
appendonly yes              //启用aof持久化方式
# appendfsync always      //每次收到写命令就立即强制写入磁盘，最慢的，但是保证完全的持久化，不推荐使用
appendfsync everysec     //每秒钟强制写入磁盘一次，在性能和持久化方面做了很好的折中，推荐
# appendfsync no    //完全依赖os，性能最好,持久化没保证
```
aof 的方式也同时带来了另一个问题。持久化文件会变的越来越大。例如我们调用incr test命令100次，文件中必须保存全部的100条命令，其实有99条都是多余的。因为要恢复数据库的状态其实文件中保存一条set test 100就够了。

为了压缩aof的持久化文件。redis提供了bgrewriteaof命令。收到此命令redis将使用与快照类似的方式将内存中的数据 以命令的方式保存到临时文件中，最后替换原来的文件。具体过程如下
* redis调用fork ，现在有父子两个进程
* 子进程根据内存中的数据库快照，往临时文件中写入重建数据库状态的命令
* 父进程继续处理client请求，除了把写命令写入到原来的aof文件中。同时把收到的写命令缓存起来。这样就能保证如果子进程重写失败的话并不会出问题。
* 当子进程把快照内容写入已命令方式写到临时文件中后，子进程发信号通知父进程。然后父进程把缓存的写命令也写入到临时文件。
* 现在父进程可以使用临时文件替换老的aof文件，并重命名，后面收到的写命令也开始往新的aof文件中追加。
需要注意到是重写aof文件的操作，并没有读取旧的aof文件，而是将整个内存中的数据库内容用命令的方式重写了一个新的aof文件,这点和快照有点类似。
* **优势**
  * 使用 AOF 持久化会让 Redis 变得非常耐久（much more durable）：你可以设置不同的 fsync 策略，比如无 fsync ，每秒钟一次 fsync ，或者每次执行写入命令时 fsync 。 AOF 的默认策略为每秒钟 fsync 一次，在这种配置下，Redis 仍然可以保持良好的性能，并且就算发生故障停机，也最多只会丢失一秒钟的数据（ fsync 会在后台线程执行，所以主线程可以继续努力地处理命令请求）。
  * AOF 文件是一个只进行追加操作的日志文件（append only log）， 因此对 AOF 文件的写入不需要进行 seek ， 即使日志因为某些原因而包含了未写入完整的命令（比如写入时磁盘已满，写入中途停机，等等）， redis-check-aof 工具也可以轻易地修复这种问题。
Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写： 重写后的新 AOF 文件包含了恢复当前数据集所需的最小命令集合。 整个重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。
  * AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松。 导出（export） AOF 文件也非常简单： 举个例子， 如果你不小心执行了 FLUSHALL 命令， 但只要 AOF 文件未被重写， 那么只要停止服务器， 移除 AOF 文件末尾的 FLUSHALL 命令， 并重启 Redis ， 就可以将数据集恢复到 FLUSHALL 执行之前的状态。

* **劣势**
  * 对于相同的数据集来说，AOF 文件的体积通常要大于 RDB 文件的体积。
  * 根据所使用的 fsync 策略，AOF 的速度可能会慢于 RDB 。 在一般情况下， 每秒 fsync 的性能依然非常高， 而关闭 fsync 可以让 AOF 的速度和 RDB 一样快， 即使在高负荷之下也是如此。 不过在处理巨大的写入载入时，RDB 可以提供更有保证的最大延迟时间（latency）。
  * AOF 在过去曾经发生过这样的 bug ： 因为个别命令的原因，导致 AOF 文件在重新载入时，无法将数据集恢复成保存时的原样。 （举个例子，阻塞命令 BRPOPLPUSH 就曾经引起过这样的 bug 。） 测试套件里为这种情况添加了测试： 它们会自动生成随机的、复杂的数据集， 并通过重新载入这些数据来确保一切正常。 虽然这种 bug 在 AOF 文件中并不常见， 但是对比来说， RDB 几乎是不可能出现这种 bug 的。

### # 主从复制
**全量同步**
Redis全量复制一般发生在Slave初始化阶段，这时Slave需要将Master上的所有数据都复制一份。具体步骤如下： 
-  从服务器连接主服务器，发送SYNC命令； 
-  主服务器接收到SYNC命名后，开始执行BGSAVE命令生成RDB文件并使用缓冲区记录此后执行的所有写命令； 
-  主服务器BGSAVE执行完后，向所有从服务器发送快照文件，并在发送期间继续记录被执行的写命令； 
-  从服务器收到快照文件后丢弃所有旧数据，载入收到的快照； 
-  主服务器快照发送完毕后开始向从服务器发送缓冲区中的写命令； 
-  从服务器完成对快照的载入，开始接收命令请求，并执行来自主服务器缓冲区的写命令；
[主从复制原理](https://www.cnblogs.com/kevingrace/p/5685332.html)
[Sentinel哨兵模式](https://www.cnblogs.com/jaycekon/p/6237562.html)



> todo
> redis IO模型
> redis能做消息队列 优点缺点  
    * 无法使用消息队列的很多特性
    * 数据丢失无法解决
    * 无法知道消息的处理结果，如果处理失败，补救的防止只能是重新丢回redis消息队列
> 分布式锁优点缺点
> 主从 数据怎么同步    SLAVEOF 127.0.0.1:6379
> 数据结构的实现原理  
> 集群和分布式集群的结构