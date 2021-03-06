# 数据类型
Redis的键值对支持多种数据类型，如此就必然需要对键值的类型进行多态处理。Redis自身建立了一个类型系统，在该类型系统中有字符串、哈希表、列表、集合和有序集等类型。
#### 1 对象处理机制  
由于键所对应值的类型有多种形式，比如：集合类型的底层实现可以是字典和整形集合，所以Redis对键进行操作的时候不仅要检查键的数据类型，还要对值进行多态处理。为了解决上述问题，Redis构建了自己的类型系统，这个系统的主要功能包括：
* redisObject 对象。  
* 基于redisObject 对象的检查。
* 基于redisObject 对象的显示多态函数。
* 对redisObject 进行分配、共享和销毁的机制。  
##### 1.1 redisObject对象
redisObject的属性有：类型、对齐位、编码方式、LRU时间（Least Recently Used：最近最少使用）、引用计数、指向对象的值。其中较为重要的是：类型、编码方式、指向对象的值。
* 类型：记录了对象所保存的值的类型。
* 编码方式：记录了对象所保存的值的编码。
* 指向对象的值：指向实际保存值的数据结构，这个数据结构由类型和编码方式共同决定。
##### 1.2 命令的类型检查
有了redisObject以后，处理执行数据类型的命令时，对类型和编码的检查就会方便很多：

1、根据给定的key，在数据库字典中查找与之相对应的redisObject，如果没有找到，就返回null。

2、检查redisObject的Type属性是否和执行命令所需的属性一致，如果不一致则返回wrongType。

3、根据redisObject的encoding属性所指定的编码，选择合适的编码函数来处理底层的数据结构。

4、返回数据结构执行操作命令后的结果。

##### 1.3 对象共享
在Redis中有许多常用的对象，比如命令的返回值：OK、ERROR、WRONGTYPE和一些小范围的整数。为了避免重复分配内存导致降低效率，Redis在内部使用了flyweight模式：通过预先给这些常见的值分配内存，让多个数据结构都可以访问这些对象。

注意：共享对象只能被字典和双端链表这类带有指针的数据结构使用，不能被整数集合、压缩列表这种存放字面量的数据结构使用。
##### 1.4 引用计数以及对象的销毁
当redisObject用作数据库的键或者值，而不是用来存储参数的时候，对象的生命周期非常长，Redis系统使用了引用计数机制来负责对象的维持和销毁:
* 每个redisObject结构都带有一个refcount属性来记录该对象被引用的次数。
* 当创建一个新的redisObject的时候，refcount设置为1。
* 当对一个对象进行共享的时候，Redis将这个对象的refcount + 1。
* 当取消对这个对象的共享或者使用完这个对象以后，Redis将这个对象的refcount - 1。
* 当一个对象的 refcount = 0 的时候，Redis系统将该对象销毁。

#### 2 字符串
字符串是Redis中应用最广泛的数据结构，数据库中的键和输入的命令都是用字符串进行存储。

字符串的编码类型有两种：REDIS_ENCODING_INT 和 REDIS_ENCODING_RAW：
* REDIS_ENCODING_INT使用long类型来保存long类型值。
* REDIS_ENCODING_RAW 使用sds（简单动态字符串）来保存其他类型的值。

Redis系统给新创建的字符串默认使用REDIS_ENCODING_RAW编码，当字符串作为键或者值保存到数据库中的时候才会转化为REDIS_ENCODING_INT编码。

Redis字符串的命令都是以包装简单动态字符串结构的操作函数来实现的。
#### 3 哈希表
哈希表的编码格式：字典和压缩列表。
* 使用字典作为哈希表的编码格式时，程序将哈希表的键存储为字典的键，将哈希表的值存储为字典的值。字典的键和值都使用字符串对象。
* 使用压缩列表作为哈希表的编码格式时，程序将哈希表的键和值一同推入压缩列表，从而形成保存哈希表所需的键-值对结构。

编码的选择：当创建哈希表的时候，Redis系统默认使用压缩列表作为哈希表的编码类型，只有当出现以下两种情况的时候，哈希表的编码类型才会转换成字典：
* 哈希表中的键或值的长度大于server.hash_max_ziplist_value （默认值为64）。
* 压缩列表中的节点数量大于 server.hash_max_ziplist_entries （默认值为512）。
#### 4 列表
列表的编码格式：压缩列表和双端链表。

编码的选择：当创建列表的时候，Redis系统默认使用压缩列表作为列表的编码类型，只有当出现以下两种情况的时候，编码类型才会转换成双端链表：
* 往列表中加入一个新的字符串值，并且该字符串的长度大于server.hash_max_ziplist_value （默认值为64）。
* 压缩列表中的节点数量大于 server.hash_max_ziplist_entries （默认值为512）。

阻塞原语：BLPOP、BRPOP和BRPOPLPUSH三个命令都可能造成客户端被阻塞，这三个命令被称为阻塞原语。

阻塞：当一个阻塞原语的处理目标为空键时，执行该阻塞原语的客户端会被阻塞。

阻塞原语并不一定会造成阻塞，只有当列表是空列表的时候，阻塞原语才会阻塞客户端，如果被处理的列表不是空列表就执行无阻塞版本的BLPOP、BRPOP 和 BRPOPLPUSH 命令。

阻塞的步骤：
* 将客户端置为 “ 正在阻塞 " 状态。
* 将客户端的信息存放到server.db -> blocking_key字典中。
* 继续维持服务器和客户端之间的通信，但是不再向客户端发送任何信息，从而阻塞客户端。

解除阻塞：
阻塞的客户端的信息被存储在字典中，字典的key就是客户端被阻塞的key，而字典的值由一个链表维护，可以用来存储多个客户端。被同一个key阻塞的客户端可能不只一个。

当客户端被阻塞以后，脱离阻塞状态有以下三种方法：
* 被动脱离：有其他客户端为造成阻塞的key推入了新的元素。
* 主动脱离：到达执行阻塞原语时设定的最大阻塞时间。
* 强制脱离：客户端强制终止和服务器的连接，或者服务器停机。

先阻塞先服务策略：当某一个key阻塞了多个客户端，Redis在解除阻塞的时候，最先被阻塞的客户端最先脱离阻塞状态，这种模式为先阻塞先服务模式（FBFS）。
#### 5 集合
集合的编码格式：整形集合和字典。

编码的选择：如果第一个元素是long long类型值，那么集合采用整形集合的编码格式，否则集合采用字典的编码格式。

编码的切换：当一个集合使用整形集合作为编码格式时，以下任何一个条件被满足时，集合的编码格式就会被转化为字典。
* 整形集合保存的整数值的个数超过512。
* 试图为集合中加入一个非long long 类型值。（加入一个非整数值）

当使用字典作为集合编码格式的时候，集合的元素被统一保存到字典的键中，而字典的值统一被设置为null。
#### 6 有序集
有序集的编码格式：压缩列表和跳跃表。

编码的选择：当通过add命令添加第一个元素到有序集中的时候，Redis系统通过审查第一个元素来决定创建怎么样的有序集。
如果第一个元素符合以下条件的话，就创建一个压缩列表编码格式的有序集：

* 服务器属性server.zset_max_ziplist_entries 的值大于 0 （默认为 128 ）。

* 元素的 member 长度小于服务器属性 server.zset_max_ziplist_value 的值（默认为 64）。

否则就创建一个跳跃表编码格式的有序集。

基于压缩列表的有序集：当有序集将压缩列表作为编码格式的时候，Redis系统使用压缩列表相邻的节点来分别存储有序集的member和score。多个元素之间按score从小到大排序，如果两个元素的score相同则按照字典对member进行对比的结果来排序。

基于跳跃表的有序集：当有序集将跳跃表作为编码格式的时候， Redis系统使用zset结构来存储有序集，zset使用字典和跳跃表数据结构来存储有序集元素。
使用字典结构使有序集在O(1)的时间复杂度内：
* 核查某一个元素是否存在于有序集合中

* 取出一个member对应的score值（实现zscore命令）
  使用跳跃表结构使有序集支持以下两种操作：

* 在 O(log N) 期望时间、O(N) 最坏时间内根据 score 对 member 进行定位（被很多底层
  函数使用）；

* 范围性查找和处理操作，这是（高效地）实现 ZRANGE 、ZRANK 和 ZINTERSTORE
  等命令的关键。

  同时使用字典和跳跃表能够高效地实现按照成员查找元素和按顺序查找元素两种操作。


































