# Redis

## 1. 数据结构

### 1.1. 简单动态字符串（SDS，simple dynamic string）

#### 底层结构

```C++
struct sdshdr {
  // 已使用的字符长度
	int len;
  
  // 当前buf数组未使用的字符长度
  int free;
  
  // 字符数组
  char buf[];
}
```

例如下图SDS示例，数组总长度是5（其实还有一个'\\0'，是6），全部已经使用，free是0

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210529152227074.png" alt="image-20210529152227074" style="zoom:50%;" />

*   如果buf[]长度不够怎么办？
    *   C提供的\<String.h\>/strcat函数可以往目标字符串数组后面拼接新的字符串
    *   因此只要事先判断长度，决定要不要拼接即可



SDS还做了如下优化：

*   空间预分配
    *   即在修改时，如果发现空间不够，会提前分配未使用内存到SDS
*   惰性空间释放
    *   在修改缩短字符串时，不会主动释放空间，而是只修改free和len；
    *   到有需要的时候，可以进行手动释放



#### 二进制安全

C语言中，字符串的结尾是通过 `'\0'` 来判断的，因此如果要保存如下一个字符串`Redis\0Tutorial`就会对字符串截断，只保存前半部分`Redis`



二进制安全指的是SDS不会对保存的内容有任何限制、过滤或假设。因为SDS是根据len来判断字符串结尾的，所以buf可以保存任意内容的二进制数组。



#### SDS有什么优势

*   性能上
    *   可以直接拿到字符串长度 O(1)，而C语言字符串要遍历整个数组O(n)
    *   通过空间预分配和惰性空间释放，使调用N次修改API，最多执行N次内存分配；而C语言字符串一定会执行N次
*   功能上
    *   调用API前会判断长度，因此不会溢出
    *   二进制安全，可以保存任意文本和二进制
    *   也支持部分C字符串的API

### 1.2. 双向链表

Redis链表实现如下：

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210529171120508.png" alt="image-20210529171120508" style="zoom:50%;" />

优势

*   双向：链表节点存在prev和next指针，因此获取前一个节点或后一个节点都是O(1)
*   无环：链表头head和尾tail的前一个和后一个都是NULL
*   带头、尾指针：因此查找头尾都是 O(1)
*   记录了长度：因此获取长度是O(1)
*   多态：节点value可以为任意类型

### 1.3. 字典(hash table)

Redis字典底层由hash表实现，大致如下：

![image-20210529171546122](https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210529171546122.png)

*   最左端的dict是字典对象，type和privdata用于实现泛型，rehashidx用于标示是否在进行rehash，-1表示当前dict没在进行rehash

*   中间的dictht表示的是hash table，size表示大小，used表示已节点的树目，sizemask为size掩码，用于计算索引值

*   最右侧的dictEntry是 hash table的项，其结构如下：

    *   ```c++
        typedef struct dictEntry {
        	// key
          void *key;
          
          // value
          union {
            void *val;
            uint64_tu64;
            int64_ts64;
          }
          
          // 指向下个哈希表节点，形成链表
          struct dictEntry *next;
        } dictEntry;
        ```

    *   可以看出其解决冲突策略是chaining（链表法）

*   掩码sizemask如何使用

    *   sizemask一般设为size-1
    *   计算方法: `index = hash & dict->ht[x].sizemask`

*   rehash

    *   load_facter = total_stored_key_num / total_hash_slot_num
    *   当load_factor较大时，会影响性能，就需要考虑rehash（load_factor过小，也会考虑收缩，进行rehash）
    *   dict含有两个ht数组，其中ht[1]是用于rehash时复制
    *   rehash时，新哈希表ht[1]的大小该如何确定
        *   rehash拓展：第一个大于 used_key_num*2 的 2的幂次方数（如4、8、16这类的数）
        *   rehash收缩：第一个大于used_key_num 的 2的幂次方数

*   渐进式rehash

    *   由于对规模较大的dict进行rehash需要花费大量的时间，因此redis不会一次性将一个字典rehash完
    *   渐进式流程
        *   为ht[1]分配空间
        *   修改rehashidx为0（表示开始进行rehash）
        *   rehash期间，任何对现有dict（即对ht[0]）进行的修改操作，都会同步进行到ht[1]上
        *   往ht[1]上每rehash一个entry，rehashidx就加一；rehash过程中，ht[0]表只减不增（即新增键也是保存在ht[1]上）
        *   rehash完成，将rehashidx置为-1

### 1.4. 跳表

![image-20210603001912647](https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210603001912647.png)

*   跳表是一种在节点上添加多层次辅助节点的有序链表 ，辅助节点两层间，下层和上层的节点数是2:1
*   核心：每层缩小一半的规模，从而达到O(log N)的性能
*   时间复杂度：插入/删除/搜索 平均O(log N)，最坏O(N)
*   造成最坏O(N)的原因：很难去构造一个perfect skip list
    *   因为如果是已知一个链表，那么我们只要先把它插成有序，然后再逐层向上构建辅助节点就行
    *   然而实际情况，我们无法预计元素到来的顺序，因此很难将其维护成一个完美的跳表
*   因此实际中，为了保持上层比下层规模少一半这个特性，可以考虑用随机化算法，保持平均规模在一半
*   随机化跳表：
    *   每层维护时，用一个一半一半的（50%/50%）抛硬币函数（flipcoin）来决定当前节点是否存在于再上一层中
    *   ![image-20210603002235908](https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210603002235908.png)
    *   即：上图中，维护L1层时，遍历L0层，遍历每个节点时，抛硬币，正面则维护该节点至L1层，否则跳过
*   跳表的查找
    *   从最上层开始，找到第一个cur.val < target && target < next.val的节点，往下下沉，直到找到target或遍历到null为止
*   随机化跳表的维护
    *   插入：调用跳表查找函数，找到应当插入的位置；然后new一个新节点，再开始摇硬币，正面则加一层，直到摇到反面 停止，然后串起需要维护的指针
    *   <img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210603003210999.png" alt="image-20210603003210999" style="zoom: 35%;" />
    *   删除：调用跳表查找函数，找到后将其删除，然后维护各指针（串到下一个去）
    *   可以想象就是插入9的反过程，把9拿掉，还原到插入9前的状态（因为插入前是平均perfect的，那么删除后也是平均perfect的，所以直接删就行，不用抛硬币了）
*   跳表的优势
    *   实现简单
    *   复杂度虽然与平衡树相同，但好像常数项稍小一点



跳表在Redis中的应用：

*   实现有序集合键
*   集群节点中用作内部数据结构

### 1.5. 整数集合

*   整数集合（intset）用于保存小规模的整数集合（元素无重复）
*   实现主要是一个用一个`int8_t`的数组，来**有序**地存储`int16_t`、`int32_t` 或`int64_t`的整数
*   如果当前整数格式不支持新加入的整数（如`int16_t`要保存一个`int32_t`数 65535），那么就需要对`int8_t`进行升级，即补零右移
*   升级的好处是节约内存
*   intset不支持降级
*   intset查找复杂度为O(log N)，即二分查找；插入移除操作复杂的为 O(N)

### 1.6. 压缩链表

*   传统数组的缺陷 要为每个元素分配一样大小的空间（比如1000个元素，有一个长度100，其他都是5）；传统链表的缺陷，需要花费大量空间存储指针（prev、next、value），如果元素本身非常小，却花费大量空间存储这些辅助指针，显然是不划算的

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210606092206023.png" alt="image-20210606092206023" style="zoom:50%;" />

*   压缩链表是一种由连续内存块组成的更加节省内存结构的链表结构，其格式如上图所示；压缩链表一般适用于空间占用比较小的数据的存储（如小整数、短字符串）
*   entryX即为列表节点；因为都是连续内存块，不需要像普通链表那样存储指针，所以更节省内存
*   entryX结构如下图所示
    *   <img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210606095754943.png" alt="image-20210606095754943" style="zoom:50%;" />
    *   其中previous_entry_length记录的是上一个entry的长度；encoding记录了当前entry保存的内容的格式（是整数还是字节数组）及长度；content保存了具体内容
*   遍历压缩链表的流程：
    *   通过zltail算出末尾entry的地址
    *   然后减去末尾的previous_entry_length从而得到前一个entry的地址
    *   依次方法，从末尾遍历到表头 遍历整个链表
    *   如果需要从表头往末尾遍历的话，可以用encoding判断长度
*   ziplist的查找、插入、删除的复杂度都是O(N)
*   ziplist优点是 内存利用率高且连续；缺点是增删时需要频繁申请和释放内存
*   3.2版本后出现了quicklist结构，主要是以ziplist为节点进行存储的双向linkedlist

### 1.7. 对象

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210529151017755.png" alt="image-20210529151017755" style="zoom:50%;" />

*   redis对象包含了字符串对象（string）、链表对象（list）、哈希对象（hash）、集合对象（set）、集合对象（sorted set）这五种对象，其底层实现用了之前介绍的六种结构：SDS、linkedlist、ziplist、hash table、skiplist、intset

*   对于Redis中的每个key-value对，Redis都会创建两个对象，一个表示key，一个表示value；其中key对象必定是String对象，而value对象可以是全部五种redis对象中的任意一种

*   要查询key对应的value对象类型，可以使用 `TYPE`命令

    *   ```shell
        > SADD fruits apple banana cherry
        (integer) 3
        
        > TYPE fruits
        set
        ```

*   要查询一个redis对象对应的底层数据结构，可以使用`Object Encoding`命令

    *   ```
        > SADD fruits apple banana cherry
        (integer) 3
        
        > OBJECT ENCODING fruits
        hashtable
        ```

*   列表对象

    *   满足以下两个条件时，使用ziplist；否则使用linkedlist
        *   保存的所有字符串元素的长度均小于64字节（短字符串）
        *   列表内的总元素数小于512（且数量较少）

*   哈希对象

    *   满足以下两个条件时，使用ziplist；否则hashtable
        *   保存的所有hash pair 的 key/value元素的长度均小于64字节（小对象）
        *   总hash pair数小于512（且数量少）
    *   ziplist保存hash对象时，内部结构如下：
        *   <img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210606170645026.png" alt="image-20210606170645026" style="zoom:50%;" />
        *   即key和对应的value紧挨着存

*   集合对象

    *   满足以下两个条件时使用intset；否则使用hashtable
        *   所有元素均为整数（整数）
        *   元素数量不超过512（且数量少）

*   有序集合对象

    *   满足以下两个条件时使用ziplist；否则使用skiplist+hashtable的组合
        *   元素长度小于64字节（小对象）
        *   元素数量少于128（且数量少）
    *   使用ziplist存储时，类似之前用ziplist保存hash对象的设计；score和value紧挨着；然后从左至右生序排列
        *   <img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210606171228058.png" alt="image-20210606171228058" style="zoom:50%;" />
    *   使用skiplist+hashtable组合时，skiplist用于保证有序扫描，hashtable用于O(1)时间内拿到value值
        *   其中skiplist和hashtable会通过指针共享存储的元素的内容，因此不会造成空间浪费

*   对象计数

    *   ```C++
        typedef struct redisObject {
        	// ... ...
          
          // 引用计数
          int refcount;
          
        	// ... ...
        }
        ```

    *   redis对象内会有一个内置字段用于表示其被引用的次数；当引用计数减为0时，对象会被redis回收

*   对象共享（类似常量池）

    *   Redis会在启动时创建0-9999的整数字符对象
    *   当别的对象需要使用到这些整数时，直接引用即可，而不需要新建

*   空闲时间

    *   ```C++
        typedef struct redisObject {
        	// ... ...
          
          // 空闲时间
          unsigned lru:22;
          
        	// ... ...
        }
        ```

    *   Redis对象还有一个属性用于记录最后一次访问的时间；这在之后回收内存的时候会使用到

-----------

## 2. 持久化

由于Redis是将数据存储在内存中，如果服务器宕机会导致数据丢失，因此需要将内存中的数据存储到磁盘中，这一过程叫持久化。

### 2.1. Redis 的持久化机制

*   RDB（Redis Database）

    *   RDB主要是将当前时刻的数据（key-value）保存并压缩为一个RDB文件；Redis可以通过该RDB文件，还原出当时数据库的状态

    *   可以通过配置实现自动间隔保存

        *   ```bash
            save 900 1			-- 900秒内，至少执行了1次修改，就调用BGSAVE进行RDB持久化
            save 300 10			-- 300秒内，至少执行了10次修改，就调用BGSAVE进行RDB持久化
            save 60 10000		-- 60秒内，至少执行了10000次修改，就调用BGSAVE进行RDB持久化
            ```

        *   满足任一，就会触发RDB持久化

    *   优点：

        *   只有一个dump.rdb文件，恢复快，也便于转移到别的机器进行故障恢复
        *   使用BGSAVE子进程进行保存，不会影响主进程，性能较好

    *   缺点：

        *   会丢数据，间隔时间段内的修改不会被持久化

*   AOF（Append-only File）

    *   AOF主要是将Redis每次的修改操作，添加到aof_buf再持久化至磁盘（类似MySQL binlog）
    *   通过将appendfsync设为always，可以保证Redis不丢数据
    *   AOF还可以进行定期重写（rewrite）来缩小AOF日志的规模
        *   AOF rewrite不需要读取任何过去的AOF日志，而是直接写入当前状态
            *   例如进行了一系列操作后，目前redis内有`a->12, b->"B"`，那么重写后的AOF日志直接就写入`SET a 12;  SET b "b"`，而不管之前到底做了什么操作
        *   在rewrite进程执行的同时，当前修改操作会被同时记录进一个AOF重写缓冲区，记录从rewrite开始时的修改操作；这样等rewrite结束后，合并两部分日志即可
    *   优点：
        *   将appendfsync设置为always后，不丢数据
    *   缺点：
        *   AOF文件大，恢复速度慢

*   如果RDB和AOF都配置了，重启时会优先加载AOF



### 2.2. 如何选择RDB还是AOF

*   如果纯用作缓存，即数据丢失也没有关系的话，那么其实可以不配置任何持久化
*   如果可以忍受几秒钟的数据丢失 或者 需要较快的故障恢复，那么选RDB
*   如果完全不能忍受任何数据丢失，那么选AOF

## 3. 主从复制与集群

### 3.1. 主从复制

*   主从复制主要是在主服务器发生修改时，从服务器能够复制其上的修改操作
*   Redis复制分为 同步（sync）和 命令传播（command propagate）两个阶段
    *   同步 指的是让从服务器同步至与主服务器相同的状态
    *   命令传播 指的是 在完成同步后，从服务器同步执行与主服务器相同的操作
*   Redis的同步过程分为  完整重同步（full resynchronization）和 部分重同步（partial resynchronization）两种模式（2.8之后这个命令叫PSYNC）
    *   完整重同步：主服务器创建RDB文件，并将同步期间的修改保存在缓冲区，之后一并发给从服务器
    *   部分重同步
        *   由于有些情况 从服务器落后主服务器不多，因此执行完整重同步会十分浪费
        *   实现：
            *   主从服务器都维护一个复制偏移量
            *   主服务器维护一个 固定长度（fixed-size）的先进先出（FIFO）的命令缓冲区
            *   如果需要进行同步的从服务器的 复制偏移量offset不在这个FIFO缓冲区内，那么进行完整重同步；否则就可以进行部分同步
            *   此外，从服务器会在初次和主服务器同步时，保存主服务器的ID；当从服务器再次重启进行同步时，需要将之前保存的ID发给当前的主服务器；当前主服务器在收到ID后，决定是进行full resynchronization还是partial resynchronization（相同则部分，不同则进行完整同步）
*   主从复制流程
    *   从服务器执行SLAVEOF，连上主服务器
    *   建立Socket
    *   从节点发送PING，检查连接
    *   完成身份认证，发送端口信息
    *   开始同步
    *   开始命令传播
    *   命令传播阶段会每秒进行心跳检测，检测连接和命令丢失（通过偏移）

### 3.2. 哨兵（sentinel）

#### 3.2.1 哨兵的主要功能

*   检测主节点、从节点是否正常
*   在主节点宕机时，提供自动的故障转移
*   作为Client连接的节点，这样可以避免主节点切换后，修改ip配置
*   服务器出故障时，可以调用API通知集群管理者

#### 3.2.2 监控主从服务器

监控过程：

*   Sentinel在启动的时候会读入用户配置（监视的主服务器列表），为每个要监控的主服务器创建实例（sentinelRedisInstance），并与主服务器创建发布（publish）连接和订阅（subscribe）连接；发布连接用于发送命令，订阅连接用于接受消息（使用channel  `__sentinel__:hello`）
*   Sentinel通过向主服务器发送INFO命令来获得其所有从服务器地址，并为其所有从服务器创建监控实例，发布连接和订阅连接
*   Sentinel还会和其他Sentinel节点建立发布连接
    *   为什么不和其他sentinel建立订阅连接？
        *   因为主从服务器在回复INFO命令时，会带上发出INFO请求的sentinel信息，因此sentinel可以通过主从服务器的订阅连接获取其他sentinel节点的信息



默认情况下，sentinel会以每秒一次的频率向与它建立了发布连接的实例（主、从、其他sentinel）发送PING命令，根据其回复来判断是否在线。

*   主观下线：指如果实例在`down-after-miliseconds`时间内，未回复，那么会将其标记为主观下线
*   客观下线：为确定其是否真的下线了，sentinel还会向其他sentinel询问其是否真的下线了
    *   使用SENTINEL命令（不同于发布连接）向其他sentinel节点询问目标服务器是否下线了；如果得到的回复，超过配置的quorum，那么就将其标记为客观下线



主服务器被标记为客观下线后，Sentinel就会开始进行故障转移。

#### 3.2.3 Leader Sentinel选举

故障转移前，需要选举出一个leader sentinel来选出新的主服务器

选举流程类似RAFT协议：

*   每个发现客观下线的sentinel都会发送SENTINEL 命令给其他sentinel，命令目标sentinel将其设置为leader
*   sentinel一旦被设置了leader sentinel信息，就会拒绝其他sentinel发起的选举请求（先到先得）
*   当有超过半数的sentinel同意了，发起选举的sentinel就会被设为leader
*   如果规定时间内没有成功，会发起一段timeout，重新进行选举



为什么需要选举leader sentinel？

*   保证一致性，只有一个sentinel去完成故障转移

#### 3.2.4 故障转移

在选出leader sentinel之后，leader sentinel会发起故障转移，选出新的主服务器：

*   从剩下的从服务器中选出一个新的主服务器
*   向被选中的服务器发送SLAVEOF NO ONE命令（将其设为主服务器）
*   通过发布订阅通道，将主服务器新的配置下发给其他从服务器
*   向从服务器发送`SLAVEOF 新主服务器`命令，使从服务器复制新的主服务器
*   当所有从服务器均开始复制新主服务器时，结束此次故障转移



选新主服务器的规则：

*   先筛除下线断线的、最后一次PING大于5秒的服务器
*   再筛除与失效主服务器连接断开的时长超过 down-after 选项指定的时长十倍的从服务器（即保证落后原主服务器不那么多）
*   再根据复制偏移量排序，选出优先级最高的服务器



### 3.3 Redis Cluster集群

主从复制和哨兵解决了Redis的高可用问题，然而我们还需要一种方法来应对 单节点的内存容量不够保存我们的数据的情况（扩容）。

Redis Cluster通过分片（sharding）来进行数据共享（即把数据分摊到多台服务器，从而扩增容量），同时提供了sentinel提供的同步复制和故障转移等功能。（即分布式数据库）

#### 其他集群方案

*   twemproxy（基于代理的分片）
    *   大概概念是，以一个代理的身份接收请求并使用一致性hash算法，将请求转接到具体redis，将结果再返回twemproxy
    *   缺点：存在单点压力；redis节点数改变时，一致性hash算法的分配也会随之改变，但twemproxy无法自动转移其中的数据
*   另外一种方案即是 基于客户端的分片，
    *   即在 客户端计算hash进行分片
    *   缺点：不能动态调整分片，集群一变动，所有客户端都要变动，维护麻烦

#### 一致性hash算法

一致性hash主要解决的是传统hash在hash table size变化的时候，需要重新映射全部键的问题；在分布式系统中，这可能意味着需要移动大量数据；一致性hash就是致力于在加减节点时，能够最少移动数据



<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210611123452354.png" alt="image-20210611123452354" style="zoom:67%;" />

基本实现

*   将整个hash值空间构成一个圆环，例如从0到2^32-1
*   选取每台机器的ip、hostname之类的信息，进行hash，得到的hash值即是该台机器的位置
*   之后存储数据时，首先对其key进行hash，得到的hash值在环上的后继机器，即为其应该存放的机器
*   增加和删除机器时，只需要维护变动机器及其两端服务器即可
*   为消除机器节点过少导致的不均衡的问题，可以考虑引入虚拟节点，即为每个机器节点创建K个虚拟节点（例如"127.0.0.1"使用"127.0.0.1#VN1", "127.0.0.1#VN2", "127.0.0.1#VN3"为key创建3个虚拟节点）



TreeMap实现如下：

```java
class Shard<S> { // S类封装了机器节点的信息 ，如name、password、ip、port等

    private TreeMap<Long, S> nodes; // 虚拟节点
    private List<S> shards; // 真实机器节点
    private final int NODE_NUM = 100; // 每个机器节点关联的虚拟节点个数

    public Shard(List<S> shards) {
        super();
        this.shards = shards;
        init();
    }

    private void init() { // 初始化一致性hash环
        nodes = new TreeMap<Long, S>();
        for (int i = 0; i != shards.size(); ++i) { // 每个真实机器节点都需要关联虚拟节点
            final S shardInfo = shards.get(i);

            for (int n = 0; n < NODE_NUM; n++)
                // 一个真实机器节点关联NODE_NUM个虚拟节点
                nodes.put(hash("SHARD-" + i + "-NODE-" + n), shardInfo);

        }
    }

    public S getShardInfo(String key) {
        SortedMap<Long, S> tail = nodes.tailMap(hash(key)); // 沿环的顺时针找到一个虚拟节点
        if (tail.size() == 0) {
            return nodes.get(nodes.firstKey());
        }
        return tail.get(tail.firstKey()); // 返回该虚拟节点对应的真实机器节点的信息
    }
}
```

#### Redis集群分片/hash slot

Redis集群通过分片（sharding）的方式来将key-value数据保存在集群中。

*   集群本身被划分为16384个槽（$2^{14} = 16384$）；当16384个槽都有服务器进行处理时，集群正常；否则集群将处于不可用状态
*   集群中每台redis实例会被指派一定的slot；例如4台机器，redis0 -> `[0, 5000]`, redis1 -> `[5001, 10000]`, redis2 -> `[10001, 15000]`, redis3 -> `[15001, 16384]`
    *   指派可以手动指定，也可以通过redis-cli工具进行自动指定
    *   每个节点都为master节点，且会记录并广播其对应的slot范围的字典
*   每个数据在计算 crc16(key) & 16383 后，查询其所属的节点
    *   如果是数据就在本节点本地，那就直接操作并返回结果给Client
    *   否则，将会返回MOVED并带上数据所属节点的地址信息；Client再收到MOVED错误后，会自动切换socket（Client连接集群时一开始就会对多个节点建立socket）重新发送请求



槽指派数组（因此可以是不连续的）：

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210611182705371.png" alt="image-20210611182705371" style="zoom:25%;" />



#### Redis集群如何实现高可用

![image-20210611182447794](https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210611182447794.png)

*   会为每个master节点配备1到N个从节点（图中的最中间四台为master）
*   redis集群无法保证强一致性：例如client写入master 1，返回OK，然而master 1的从节点还没完成同步复制，master 1就挂了，就会丢数据
*   三主三从结构
    *   最好至少有6台机器，这样master、slave各一台
    *   如果只有3台机器，那么就每台上部一个master一个slave，但是slave同步的master必须是其他机器的master节点

-----------

## 4. 事务

*   Redis事务是指 一系列命令将会被一起连续集中执行（区别于传统意义上的事务：把多个操作看成一个整体，要不全都执行，要不全都不执行）
*   使用 MULTI、EXEC、WATCH等命令完成	
    *   MULTI标志事务开始
    *   EXEC标志事务执行
    *   WATCH用于在MULTI前使用，表示监视某个key，如过在事务执行期间该key遭遇修改，那么事务将被拒绝执行并返回空
    *   DISCARD表示清除事务队列，并退出事务
*   MULTI和EXEC间的命令都会被放入一个FIFO队列，在EXEC时依序执行
*   事务中出错
    *   语法错误：那么直接在入队（EXEC前）的时候就会报错；然后在EXEC时被服务器拒绝这个命令队列，一个命令都不执行
    *   运行时错误：除了出错语句，都执行（不支持回滚）
*   事务不支持回滚
    *   Redis作者认为事务出错只有可能是开发者的编程错误，在生产上不可能出现，所以没必要支持
*   事务ACID满足与否
    *   原子性：不满足，因为发生错误仍会执行下去（即不会回滚）
    *   一致性：满足，因为无论如何执行，都不会对redis数据库造成非法或无效的操作导致不一致
    *   隔离性：满足，由于单线程，显然是可以满足隔离性的
    *   持久性：由持久化模式决定，如果是AOF并且appendsync设为always，那就是满足
*   其他实现方法
    *   lua脚本

----------------

## 5. 键过期和内存淘汰

### 5.1. Redis键过期如何实现？

*   总的来说，过期键一般有三种删除策略：
    *   定时删除
        *   即设置一个timer，在键过期时进行回调通知
        *   优点是 能够及时释放无用内存（内存友好），缺点是 占用大量计算资源（CPU不友好）
    *   惰性删除
        *   即在每次根据key取值时，判断是否过期，如果过期，返回空并删除该键
        *   优点是 几乎不额外在键过期上花时间（CPU友好），缺点是 过期对象的内存不能及时释放（内存不友好）
    *   定期删除
        *   即隔一段时间对数据库内的键进行扫描，扫到过期的就删除
        *   前两种方法的折中手段，难点在于如何设置扫描间隔
*   Redis采用 惰性删除 和 定期删除 结合的方式进行过期键删除
    *   其中，定期删除的部分采用如下逻辑
        *   在设置的最长定期删除时长内，随机抽取键，进行过期检查，如果过期就删除
        *   由于设置了最长定期删除时长，因此定期扫描的时间是可控的

### 5.2. 内存淘汰

当Redis内存使用达到maxmemory配置时，会触发内存淘汰机制，淘汰掉一些key从而获得内存。具体有如下策略：

*   `noeviction`：默认策略，内存不足时，新的写入操作会报错
*   `volatile-lru` ：删除expire set（即设置了expire值的key集合）中的一个键，尝试删除最近未使用的键
*   `volatile-ttl`： 删除expire set中的一个键，尝试删除剩余时间较短的键。
*   `volatile-random` ：删除expire set中的一个键，随机选取
*   `allkeys-lru`：删除整个key集合空间的一个键，尝试删除最近未使用的键
*   `allkeys-random` ：删除整个key集合空间的一个键，随机选取



老版本有使用虚拟内存，即把最近最久未使用的key置换进swap区；但是好像性能较弱，被废弃（deprecated）了



---------

## 6. 线程模型

### 6.1. Redis的线程模型

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210612152942769.png" alt="image-20210612152942769" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210612204554178.png" alt="image-20210612204554178" style="zoom:57%;" />

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210612204631559.png" alt="image-20210612204631559" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/wumrwds/notes/main/Redis/pics/image-20210612204906124.png" alt="image-20210612204906124" style="zoom:50%;" />

*   基于Reactor模式（event-driven）；
    *   由一个IO多路复用程序监听连接客户端的Socket
    *   当监听到Socket准备好执行应答（accept），读取（read），写入（write）等操作时，产生对应的文件事件
    *   处理事件的是一个单线程的event loop程序
        *   即一个while循环，每次通过epoll取到新事件，然后调用对应的事件handler（相当于一个函数）进行处理
        *   先处理文件事件，然后处理时间事件（serverCron事件）
        *   处理完后重新开始循环
*   多路复用和双工通信
    *   多路复用（multiplexing）：例如reactor模式里，所有Client发起请求，都会被一个线程统一处理，然后分派给handler进行处理，然后处理完后再通知进行返回；就是用一套这个处理线程，去处理多个IO事件
    *   双工（duplex）：指的是两个设备可以同时进行双向的数据传输和接收；例如两个人打电话，可以互相说话并听到对方
*   单线程的好处
    *   避免线程切换的消耗
    *   避免加锁、释放锁产生的消耗，同时也不用考虑死锁问题
*   6.0版本后出现多线程，主要思路为：
    *   Redis单线程实现的主要瓶颈在于IO读写
    *   因此多线程的思路主要是将IO读写新开一个进程，让主线程能够继续处理请求，从而提高吞吐量

-------

## 7. 缓存雪崩、击穿、穿透、预热、降级

#### 缓存雪崩

*   缓存雪崩（cache avalanche）指的是某一个时刻，key大量过期，使得大量查询跳过缓存，直接命中数据库
*   解决：
    *   设置过期时间时，随机加上一个timeout；使错峰过期
    *   热点数据缓存永不过期
    *   使用熔断机制，限流降级。当流量达到一定的阈值，直接返回“系统拥挤”之类的提示，防止过多的请求打在数据库上将数据库击垮，至少能保证一部分用户是可以正常使用，其他用户多刷新几次也能得到结果。

#### 缓存击穿

*   缓存击穿（cache breakdown）指的是某个热点key的过期，导致大量单点请求直接打到数据库
*   解决：
    *   热点数据不过期
    *   互斥锁控制读数据库和写缓存的线程数量；比如某个key只允许一个线程查询数据和写缓存，其他线程等待。这种方式会阻塞其他的线程，此时系统的吞吐量会下降

#### 缓存穿透

*   缓存穿透（cache penetration）指的是 用户请求一个不在数据库内的数据，由于其不可能在缓存中命中，会直接打到数据库；一般被黑客用作攻击手段，打爆数据库
*   解决：
    *   业务上进行控制，比如超过一些参数范围，就不查询直接返回
    *   将无效key设置一个很短的过期时间，也存在缓存中（有一定问题，如果无效key是随机的，也会打爆缓存）
    *   使用bloom filter
        *   bloom filter判断一个key如果不在缓存中，那么一定不在；但是如果判断在，可能会在
        *   因此可以用bloom filter先进行一层筛选

#### 缓存预热

*   缓存预热（cache warming）指 系统上线后，先提前将相关数据加载至缓存；避免系统一上线是一个无缓存的状态，直接被用户请求打爆
*   方案
    *   项目启动时，写逻辑加载
    *   定时脚本或任务加载

#### 缓存降级

*   缓存降级（cache downgrade）指 当访问量激增 或 服务器崩溃 影响到核心业务功能时，可以考虑对次要服务进行缓存降级，即停止次要业务的缓存功能，以保证主要业务运行
*   例如双十一高峰时，淘宝购物车无法修改地址只能使用默认地址，即当时为保证支付等业务进行，降级了这一部分功能；高峰过去，将会恢复这部分功能
*   方案：需要提前设置好开关，然后配合监控自动切换或是人工手动进行切换

-------

## 8. 高级用法

### 8.1. 分布式锁

分布式锁一般用于控制分布式系统间互斥地访问共享资源

*   例如 一个购买操作，可能需要分为 查库存和修改库存两个操作；这样如果两个用户同时发起请求，可能会造成超卖的现象

### 8.2. Redis实现

*   setnx + expire实现

    *   即 先setnx，如果设置成功，那么接着设置锁的过期时间（使用expire），设置不成功，则表示锁已被占用，退出
    *   缺点：由于setnx和expire是两个操作，不具有原子性；可能setnx设完，应用挂了，没设置expire，那么这个锁就永不过期了

*   `set key value [EX seconds][PX milliseconds][NX|XX]` 实现

    *   其中参数 EX和PX表示过期时间，NX表示仅当不存在时设置值（即等同带过期的setnx），XX表示 仅当存在时设置值

*   以上方法在设置key时，一般会带上一个当前线程产生的UUID，这样只有当前线程能够根据key进行 del key操作解锁

*   以上方法在遇到Redis master节点宕机，slave切换的时候，可能会出现多个Client拿到锁的情况 或者 master节点脑裂，出现两个master节点的情况

    *   例如Client A set命令在master节点上执行成功，但还没有同步到slave上；此时master宕机，slave成为master后，锁失效；然后Client B 又通过set命令拿到锁
    *   为解决上述问题，Redis提出了Redlock算法

*   Redlock算法

    *   获取当前时间戳

    *   Client尝试顺序（sequentially）获取所有redis节点的锁。获取锁时设置一个极小的timeout（远小于设置的auto-release时间），timeout内没拿到，就去获取下一个
        *   比如：设置的锁自动释放（auto-release）时间是5s，那么timeout就设成5～50ms左右

    *   client通过获取所有能获取的锁后的时间减去第一步的时间，这个时间差要小于TTL时间并且至少有N/2+1个redis实例成功获取锁，才算真正的获取锁成功

    *   如果成功获取锁，则锁的真正有效时间是 auto-release时间减去第三步的时间差 的时间；比如：锁自动释放时间 是5s,获取所有锁用了2s,则真正锁有效时间为3s(其实应该再减去时钟漂移);

    *   如果客户端由于某些原因获取锁失败，便会开始解锁所有redis实例；因为可能已经获取了小于3个锁，必须释放，否则影响其他client获取锁

*   Redis分布式锁的缺陷

    *   稳定性一般；单点宕机有概率出现问题，集群环境下的redlock算法也有一些问题，比如时钟跳跃，锁生效时间可能过短等问题
    *   Redis的分布式锁需要Client不断的去尝试获取锁，耗性能

*   另一种分布式锁的实现是使用zookeeper（现在也有使用etcd的）



---------

## 其他问题

### Redis主从不一致怎么处理





#### 假如 Redis 里面有 1 亿个key，其中有 10w 个key 是以某个固定的已知的前缀开头的，如果将它们全部找出来？如果使用keys指令会有什么问题？

使用 keys 指令可以扫出指定模式的 key 列表。

redis 的单线程的。keys 指令会导致线程阻塞一段时间， 线上服务会停顿， 直到指令执行完毕， 服务才能恢复。可以考虑使用scan



#### 如果有大量的 key 需要设置同一时间过期，一般需要注意什么？

可以稍微加上一个小的随机timeout，使其不要同时过期；但是其实redis的过期策略应该可以避免这种情况



#### Redis如何实现延时队列

 

#### 使用Redis做过异步队列吗，是如何实现的



#### Redis回收进程如何工作的？



#### Redis实现附近的人



#### Redis实现bloom filter

Bloom filter 就是用k个hash函数，将一个数据散列到一个bitmap上的多个点；这样就可以进行判断



#### Redis 使用 HyperLogLog 来进行 UV 统计



#### Redis为什么快



#### 常见性能问题和解决方案



#### 读写分离方案