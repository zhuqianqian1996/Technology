##  内部数据结构
### 1 简单动态字符串（sds）
&emsp;&emsp; Redis中所有的字符串都是简单动态字符串，简单动态字符串结构内部由len、free、buf[]三个属性组成，
len表示当前字符串的长度，free表示未使用的空间大小，
buf[]是实际上存储字符串的区域。这样做的好处：  

   * 可以高效的执行长度计算，简单动态字符串内部的len属性可以在O(1)时间复杂度内计算出字符串的长度。<br>
   * 优化了追加的操作，通过在buf中分配多余一倍的内存空间，在下一次执行append操作的时候，如果追加的字符串长度小于剩余内存空间的长度，就无需再重新为buf分配内存，减少了内存分配的次数，提高了追加操作的效率。但是这也造成了内存空间的部分冗余，并且这部分内存不会被释放。
    
### 2 双端链表
#### 2.1 双端链表在Redis中的应用
*  实现Redis的列表类型；(双端链表or压缩列表)
* 被Redis中的某些应用模块使用： <br>
    (1) 事务模块使用双端链表来保存事务中按顺序输入的命令。<br>
    (2) 服务器模块使用双端链表来存储多个客户端。<br>
    (3) 订阅发送模块使用双端链表来保存订阅模式中的多个客户端。<br>
    (4) 事件模块使用双端链表来保存时间事件。  
	
#### 2.2 双端链表中的属性
&emsp;&emsp;*head表头指针，*tail表尾指针，len节点数量，复制函数，释放函数，比对函数。<br>
&emsp;&emsp;节点中的函数用于处理值的复制，释放和比对匹配的操作。表头指针和表尾指针可以高效的实现LPUSH,RPOP等操作。Len字段能够在O(1)时间内计算链表的长度，保证了LLEN命令的效率。
#### 2.3 迭代器：
&emsp;&emsp;Redis为双端列表提供了一个迭代器，迭代器中有*next指针和direction变量，*next指针指向列表中下一个访问的元素，direction用来标识迭代的方向。

### 3 字典
&emsp;&emsp;字典是一种抽象数据结构，由一集键值对（key-value pairs）
组成，各个键值对的键各不相同，程序可以将新的键值对添加到字典中，
或者基于键进行查找、更新或删除等操作。  
#### 3.1 字典的应用
&emsp;&emsp;字典的应用有两种：实现数据库键空间；作为hash类型键的一种底层实现。
* 键空间：Redis中键值对的数据被保存在一个字典中，每一个Redis数据库有一个字典，这个字典就是键空间。当程序存储一组键值对数据时，就会将键值对保存到字典中，当程序删除一组键值对数据时，就会从字典中移除该键值对。
* 作为hash类型键的底层实现：hash类型键的底层有两种实现方式：
	  压缩列表和字典。一般情况下默认是压缩列表(压缩列表比字典更节省内存),
	  当到了特殊的情况下hash类型的键就由字典来实现。
#### 3.2 字典的实现
&emsp;&emsp;Redis使用两个哈希表来作为字典的底层实现，其中0号哈希表用于主要存储字典中的数据，当需要rehash的时候才会使用1号哈希表。字典中的哈希表使用链地址法来解决哈希冲突。
#### 3.3 创建新字典
&emsp;&emsp;创建新字典的时候，哈希表并没有被分配任何空间，在第一次向字典中存放数据的时候才会为第0号哈希表分配空间，当出现哈希冲突的时候，使用链地址法解决哈希冲突，如果需要进行rehash操作时才会为第1号哈希表分配空间。
#### 3.4 添加键值对到字典
&emsp;&emsp;将键值对添加到字典中，根据字典此时的状态需要进行一系列复杂的操作：
* 添加键值对时字典还没有初始化，则需要对字典进行初始化，即初始化第0号哈希表。
* 如果发生了哈希碰撞则需要处理碰撞。
* 如果添加元素满足了rehash的条件，就执行rehash相关的操作。  
处理完这三个操作以后，元素才能被添加到字典中。
#### 3.5 添加新元素到空白字典
&emsp;&emsp;第一次添加新元素到空白字典中时，程序会为第0号哈希表分配4个单位的内存空间，然后将新元素插入到第0号哈希表中。
#### 3.6 添加新键值对时发生碰撞处理
&emsp;&emsp;当添加键值对发生碰撞时，程序会采用链地址法在第0号哈希表发生碰撞的元素后面链上新的元素，这样就解决了哈希冲突的问题。
#### 3.7 添加新键值对引发rehash操作
&emsp;&emsp;当添加新键值对过多的时候，会产生大量哈希冲突，
此时哈希表的查询效率就和在链表中查询效率相似，
如此一来就会降低Redis的查询效率。  
&emsp;&emsp;此时程序会对哈希表进行扩容操作：
在不修改任何键值对的情况下，
将键值对的数量 : 哈希表的长度尽量维持在1:1。
下面两个条件中满足任一个条件，就会执行rehash操作：<br>
* 强制rehash : 哈希表上的键值对数量 :  哈希表的长度 > 变量dict _force_resize_ratio（目前版本dict _force_resize_ratio = 5）时，Redis会对哈希表强制rehash。
* 自然rehash： 哈希表上的键值对数量 :  哈希表的长度 >  1 并且 dict_can_rehash = true时，就可以进行rehash操作。  

#### 3.8 rehash操作步骤
* 为第1号哈希表分配更多的内存空间（大小最少是第0号哈希表的两倍）。  
* 将第0号哈希表上的键值对迁移到第1号哈希表中。
* 释放第0号哈希表的空间，交换两张表的序号，此时就可以在不修改键值对的前提下完成rehash操作。
  
#### 3.9 渐进式rehash操作
&emsp;&emsp;由于一次性rehash操作产生时延，并且阻塞Redis服务器降低Redis的执行效率，所以Redis采用了渐进式rehash操作，将rehash操作分解为多个步骤进行。对数据库字典和哈希键建立索引并根据索引进行rehash操作；设置rehash时间，在指定的毫秒数内尽量去执行rehash操作，从而加速了字典的rehash进程。
#### 3.10 字典的收缩
&emsp;&emsp;字典的收缩也是通过rehash来实现，当哈希表可用节点数远远大于已用节点数的时候，可以通过rehash操作实现字典的收缩，为第1号哈希表分配较小的内存空间，然后将第0号哈希表中的键值对迁移到第1号哈希表中，最后交换两张哈希表的序号，就可以实现字典的收缩。
  <br>**注意**：字典收缩和字典扩展的区别：字典收缩是由程序手动执行的，字典扩展是由哈希表自动触发的。 
#### 3.11 字典的迭代
&emsp;&emsp;字典的迭代实际上就是对字典中哈希表的迭代，
在没有进行rehash操作时，Redis迭代第0号哈希表；
如果正在进行rehash操作时，就会先迭代第0号哈希表，
然后迭代第1号哈希表。  
### 4 跳跃表
#### 4.1 跳跃表的组成部分：
* 表头：负责维护跳跃表的节点指针。
* 跳跃表节点：保存着元素值，以及多个层。
* 层：保存着指向其他元素的指针。高层的指针越过的元素数量大于等于低层的指针，为了提高查找效率，程序总是从高层开始访问，然后随着元素值范围的缩小，慢慢降低层次。
* 表尾：全部由NULL组成，表示跳跃表的表尾。   

#### 4.2 跳跃表在Redis中的实现：
* 跳跃表在Redis中可以允许重复的score值，多个member的score值可以相同。
* 进行对比操作时，需要检查member-score组，否则无法判断一个元素的身份。
* 每个节点都有一个高度为一层的后退指针，表示从表尾向表头方向迭代：当执行zrevrange或zrevrangebyscore这类逆序处理有序集操作时，可以用到这个属性。
#### 4.3 跳跃表的应用：
&emsp;&emsp;跳跃表在Redis中是有序集合的底层数据结构，当创建一个有序集合的时候，Redis会以score值为索引，对有序集元素进行排序：分数高的元素在高层节点中，分数低的元素在低层节点中。
