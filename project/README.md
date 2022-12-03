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