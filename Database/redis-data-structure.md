>对数据结构部分实在不太感冒，，，比较好奇持久化、主从&哨兵&集群、网络部分的实现，所以随便记点假装自己看过，，，差不多得了😅

## SDS

### 问题

C 原生字符串：

* 不是二进制安全
* 读相关操作（e.g. 获取长度）复杂度 O(N)
* 有溢出风险（e.g. 拼接字符串 s1， s2 时，如果没有为 s1 申请足够内存，导致 s1 后方区域内存被覆盖）
* 朴素的分配策略在面对频繁修改的 workload 时影响性能

### 解决

* 增加 `len` 字段标识结尾而非 `'\0'`，实现二进制安全
* 同上，通过增加长度属性把复杂度降到 O(1)
* 同上，增加长度属性约束可用范围，防止缓冲区溢出
* 减少内存分配与销毁开销，甚至尽可能避免这些操作
  * 空间预分配：其实就是根据具体情况多分配一些内存来降低分配频率
  * 惰性释放空间：缩短时仅将 `len` 值减小而不会真正释放内存，这部分内存可以直接被复用

### 实现

* 实际分配 sds 内存时长度 +1 为结尾的 `'\0'` 预留空间，因而 `buf[]` 的实际长度为 `alloc+1`

  `sh = s_malloc(hdrlen+initlen+1);`

* 将 sds 定义为 char * 的别名从而使其兼容 C 风格字符串处理函数

  `typedef char *sds;`

* sdshdr 结构体定义中的 `buf[]` 用了一个柔性数组的 trick，这样它的地址和结构体连续，可以直接通过首地址偏移得到结构体的首地址，从而得到所有成员变量

* `__attribute__ ((__packed__))` 取消内存对齐，进一步压榨空间，同时便于指针操作

* 必要时 sds 也提供直接释放内存的方法；惰性释放时用 `sdsclear`

  ```c
  void sdsfree(sds s) {
      if (s == NULL) return;
      s_free((char*)s-sdsHdrSize(s[-1]));
  }
  void sdsclear(sds s) {
      sdssetlen(s, 0);
      s[0] = '\0';
  }
  ```

* 当需要扩容时（e.g. `MakeRoomFor`）往往会采用按需动态分配的策略，且对上层透明

* 函数复用比较频繁，接口设计方面值得学习

## 跳表

### 问题

数组插入删除慢，链表查询慢，红黑树实现复杂 => 跳表 => 效率堪比红黑树，实现简单得多

### 解决

本质上跳表还是典型的空间换时间的做法，通过建立多层索引来跳过有序集合中的部分数据，层数越高跳的越快，占用空间就越多；并且根据跳表的特点我们知道越高层的索引节点越少，这样数据量大到一定程度时跳表的查询复杂度近似为 O(logN)。Redis 中跳表主要还是用来实现有序集合，其他地方不怎么用到，

### 实现

实现其实还是挺简单的，随便写一写逻辑

* 插入：

  * 查找插入位置

    一个循环找到每层中被插入节点的前一个节点，以及当前层从头节点到前一个节点经历的步长

  * 调整跳表高度

    因为新节点的高度是随机取的，如果大于当前跳表高度就需要调整

  * 插入节点

    修改插入节点前后节点的信息

  * 调整 `backward`

    让新节点后退指针指向之前找到的前一个节点，如果新节点是最后一个节点那就修改尾节点指针

* 删除：

  * 查找节点位置
  * 调整前驱结点的 `backward` 和 `span`，还有跳表本身的属性，同时也要注意尾节点的 corner case

差不多得了，跳表这实现其实和链表大同小异，只是加了点东西用来建索引。会写链表就会写跳表（迫真）

## 压缩列表

### 问题

没什么问题，单纯为了节省空间，采用了特殊的编码机制使得可以存储大小不同的元素，并通过内存偏移双向访问，避免存指针占用的内存，以及表头表尾的操作（增删）也是 O(1) 的

### 解决

感觉压缩列表真有种畸形（？）的意味，为了极端的省内存不惜冒 O(N) 的插入删除复杂度、甚至可能连锁更新的风险，但是实践证明在数据量不大的时候这些并不能影响什么性能，所以压缩列表被用作很多 Redis 数据结构的底层模块，然后在数据量达到阈值时转换成别的底层实现（e.g. 跳表、双向链表）

### 实现

编码方式和内存布局可以看看源码上方的注释，写的挺清楚

解码规则不想复读了

主要看看连锁更新：其实就是因为每个节点要记录一个 `<prevlen>`，假如你改了当前节点的长度并且恰好触发了大于 254 的条件，那记录这个值的空间也要发生变化，进而导致后置节点的 `<prevlen>` 也发生变化，然后在一个巧妙的情况下就能让一堆节点像多米诺骨牌一样触发连锁更新。听起来很恐怖，但是这种情况很少，所以对性能没什么大影响。

## 字典

### 问题

~~C 语言没有内置字典~~

### 解决

* 底层实现为哈希表
* 存两个哈希表，一个只在 rehash 的时候用
* 链地址法解决 colision
* rehash 时可能阻塞主线程 => 渐进式 rehash，均摊操作量

### 实现

* 这个掩码优化取余速度还是挺秀的
* 指纹类似于 checksum，是用来确保非安全迭代器使用前后字典数据安全的
* 稍微总结下迭代器：
  * 有安全迭代器 => `iterators++` => 不执行单步 rehash
  * 无安全迭代器 => `iterators == 0` => 执行单步 rehash
  * 安全迭代器可以进行修改，非安全迭代器只能遍历
  * 对字典的任何修改都会调用 `dictRehashStep` 进行单步 rehash，从而导致指纹改变
  * `keys` 命令对数据库进行全遍历，用的是安全迭代器所以无法 rehash，进而使得 redis 短暂的不可用，于是乎 2.8.0 开始提供了 `scan` 命令进行间断遍历
  * //TODO：等后面搞持久化的时候再回来看看，这玩意暂时还整不太明白应用场景

## 整数集合

懒得套格式了

* 和压缩列表作用差不多，都在元素量不多时尽可能压榨内存，不过整数集合中元素是有序且相同类型的
* 有序 => 可以二分 => 查询复杂度 O(logN)
* 相同类型 => 便于直接通过索引取值
* 「升级」=> 根据新插入的元素决定是否要提升类型大小
* 升级的话要进行一次全拷贝
* 还是那句话，数据量多了就没法忽视本身较高的复杂度带来的操作负载，所以仅在数据量小时用一用

## quicklist

快睡觉了，随便整点活糊弄糊弄得了

* 节省内存节省内存节省内存
* 降低修改开销降低修改开销降低修改开销
* 得有个 O(1) 的绝活（e.g. 修改头尾元素）
* 本质上还是访问效率和内存占用的 tradeoff，可以用一些奇技淫巧比如压缩来针对特定 workload 优化
* 好了quicklist 就这么结束了

## Stream

姚了我吧，我要睡觉了

//TODO：后面看到哪里用到了再补

---

publish: 2021-02-16

update: 2021-02-16