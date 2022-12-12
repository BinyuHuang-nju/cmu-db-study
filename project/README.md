## cmu-db-lab
见仓库[bustub](https://github.com/BinyuHuang-nju/bustub/tree/main)，main branch.

### Lab 0 (前缀树Trie Tree)
注意点：
  
1.	类之间的移动构造函数和右值的使用
2.	使用dynamic_cast进行派生类与基类之间的强制类型转换
3.	为保证线程安全所加的锁，这里我比较偷懒 直接使用全局锁从头锁到尾
4.	Insert、GetValue本身比较简单，比较麻烦的是Remove，我们需要自底向上递归删除节点，在实现中记录了两个bool型need_to_delete和success，前者表达子节点告诉当前节点它是否要删除子节点，后者表达该key我们是否成功删除了


### Lab 1 Buffer Pool

#### Phase 1: Extendible hash table
1.	类Bucket较为简单，我在实现时，给Find和Remove释放局部锁，Insert与锁无关，这是因为我想在类ExtendibleHashTable中先给bucket获取锁，就立马释放全局锁，再对该bucket做相关操作，从而提高hash table整体并发度。
2.	假设对全局锁的持有周期为A，对某bucket锁的持有周期为B，我希望A和B有交集，确保A到B中间没有插入其他动作(或仅插入读动作)而使获取到的bucket已经被改变；此外，A和B的交集尽量小，提高并发度；并且我们要保证所有操作持有锁的顺序皆为A->B，以防出现循环等待而死锁。
3.	类ExtendibleHashTable的Remove、Find较为简单，比较难的模块在Insert，若Insert发现该bucket已满，我们需要对该bucket做split。split规则为，自增local depth，有必要的话增大global depth并double directory(注意每个元素的指针指向)，对原bucket中的每个kv对，重新insert进不同的bucket中。
4.	上述insert中要注意的是：一，在做新的插入之前 将bucket锁释放；二，有可能某一次拆分后，原bucket中的所有kv依然路由到了同一个桶中，而当前需插入的kv也被hash到了该桶，导致还需要对该桶做拆分，所以记得在最外面套一个for循环，直至插入时bucket还未满为止。

#### Phase 2: Replacer

##### LRU Replacer  

双向链表 + hash表


##### LRU-K Replacer  

-	维护两份 双向链表 + hash表，分别存储history list(access次数小于k)的frame和cache list(access次数不小于k)的frame。  

-	偏麻烦一点的函数在于Evict和RecordAccess。Evict我们先从history list尾部开始查找第一个evictable的frame，若没找到，从cache list尾部开始查找第一个evictable的frame。
-	RecordAccess分为新加入的节点和已在list中的节点。若是新加入的节点，接入history list头部。当前frame的access次数+1后，若为k，则将其从history list中移至cache list头部，否则，将该frame移至当前list的头部。

-	关于并发锁，这里我使用了比较暴力的scoped_lock，原本希望给两个list各持一把锁，但实现过程中发现两把锁的生命周期接近，且几乎占据各个函数头尾，故采用了比较简单的加锁方式。

-	可优化的点：其他函数均为O(1)，而Evict函数需要evict出当前最旧的evictable frame，需要从尾部向前遍历，最差复杂度为O(n)。若要将其优化到O(1)，可以给两个list另外记录其中evictable的数据结构，应配套类似的 list + map的数据结构，确保Evict降到O(1)的同时，其他三个函数复杂度不变。这里感觉这种实现方式空间成本较高，内存上的放大较为明显，且代码更混乱，暂时不考虑该方案。


#### Phase 3: Buffer pool manager
-	仔细一点照着每个函数的brief写即可

-	主要注意调用disk manager的ReadPage、WritePage时机

-	注意若某page的pin count不为0时，任何时刻都不能移出bp，防止内存泄漏

-	注意pin count与调用replacer.SetEvictable之间的因果关系

-	注意NewPgImp与FetchPgImp之间的区别，一边需要调用AllocatePage()获取page id，一边需要ReadPage从disk fetch出该已有page的数据

-	注意关键数据结构之间的同步，replacer和page table、free list这三者内所表达的pages应该在任何一个函数前后都是相通的。
	-	如：函数NewPgImp、FetchPgImp中，主动换出旧page换入新page时，确保先在replacer、page table中删除旧page数据、重置frame内容，再换入新page，在replacer、page table中加入新page数据。
	-	如：函数DeletePgImp中，删除page时，需要保证page table、replacer中删除该page信息，free list中加入该frame，若为脏页还需刷盘，再重置该frame内容。  



### Lab 2 B+ tree
笔记：  

1.	多态类之间的类型转换用dynamic\_cast，不同类型的指针类型之间的转换用reinterpret\_cast。  

2.	HeaderPage in header\_page.h:
	1.	继承自Page类，故有Page的4KB data，lsn，pin count，page id等属性
	2.	是对4KB data\_ content的组织管理。data\_中，开始的4B为int型的record count，代表该page中record的数量；从offset 4b开始，后面为一个个的36b的record，36b分为32b + 4b，前32b存放该record数据，后4b存放page\_id\_t型的root\_id。

3.	对B+树的任意更新操作都有可能修改root\_page\_id，此时我们需要立刻调用BPlusTree::UpdateRootId来保证B+ tree index被持久化到磁盘上。