---
title: leveldb源码系列1-skiplist
date: 2022-09-09 10:24:02
tags: leveldb
---

本文分析的是leveldb中的跳表skip list的实现，他会把user key和user value打包成一个更大的key塞入list中
跳表的一个例子如下图


可以看到，每一个node，它都有不同的高度，且每个节点都在第0层都有出现，第0层就像最简单的链表一样，而到了上面的层数节点的个数越来越少，就像树状结构那种，跳表的许多操作都能在logn的复杂度下完成，
leveldb的主要结构包括skiplist，内部是由一系列的node构成的，他还实现了一个iterator用于遍历
接下来首先看node的实现
~~~
template <typename Key, class Comparator>
struct SkipList<Key, Comparator>::Node {
  explicit Node(const Key& k) : key(k) {}

  Key const key;

  // Accessors/mutators for links.  Wrapped in methods so we can
  // add the appropriate barriers as necessary.
  Node* Next(int n) {
    assert(n >= 0);
    // Use an 'acquire load' so that we observe a fully initialized
    // version of the returned Node.
    return next_[n].load(std::memory_order_acquire);
  }
  void SetNext(int n, Node* x) {
    assert(n >= 0);
    // Use a 'release store' so that anybody who reads through this
    // pointer observes a fully initialized version of the inserted node.
    next_[n].store(x, std::memory_order_release);
  }

  // No-barrier variants that can be safely used in a few locations.
  Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_relaxed);
  }
  void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_relaxed);
  }
   private:
  // Array of length equal to the node height.  next_[0] is lowest level link.
  // 1) 这里提前声明并申请了一个内存，用于存储第 0 层的数据，因为第 0 层必然存在数据。
  // 2) 这里的数组长度其实就是层高，假设 next_ 长度为 n，那么就会从 next_[n-1] 开始查找。
  // 3) 因为 skip list 的 level 并不会太大，使用数组存储 Node 指针的话对 CPU 内存更友好
  // https://15721.courses.cs.cmu.edu/spring2018/papers/08-oltpindexes1/pugh-skiplists-cacm1990.pdf
  std::atomic<Node*> next_[1];
};
~~~
第一部分主要是1个显式的构造函数，指定某个键并初始化key这个成员数据，然后是他的next_数组，这个数组主要是用来存放该结点的每一层的next结点的指针的，指定为1是因为必然要在第0层有该结点，接下来是他的2个无锁操作和2个不用内存屏障的操作，next这个无锁操作使用了next_这个原子对象的load函数，且指定了memory_order_acquire,那么在这个语句之前的都不会被重排到他后面了，而setnext则是store函数，指定了memory_order_release，则该语句后面的内容都不会重排到他前面去，
后面的2个则是使用了memory_order_relaxed,他只保证这条语句他是原子的，语句前后怎么重排都没有限制
接下来是一个生成新结点的函数

~~~
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node* SkipList<Key, Comparator>::NewNode(
    const Key& key, int height) {
  // 内存分配时只需要再分配 level - 1 层，因为第 0 层已经预先分配完毕了。
  char* const node_memory = arena_->AllocateAligned(
      sizeof(Node) + sizeof(std::atomic<Node*>) * (height - 1));
  // 这里是 placement new 的写法，在现有的内存上进行 new object
  return new (node_memory) Node(key);
}
~~~
第2行开头的typename是为了告诉编译器，后面这个::Node是一个类型，那么整个函数的返回值就是NOde*了，首先分配内存，然后在这个内存上placement new，调用node的构造函数了

接下来是skiplist的成员函数
第一个是生成随机层数的函数
~~~
template <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && ((rnd_.Next() % kBranching) == 0)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
~~~
首先初始化height为1，接着以1/4的概率使得while成立（在height比kmaxheight小的情况下），这样子第1层的node个数就大致是第0的1/4了，后面的层数以此类推，而用1/4这个概率貌似也是提出跳表的论文中建议的？
接下来是一个key的大小顺序的函数
~~~
template <typename Key, class Comparator>
bool SkipList<Key, Comparator>::KeyIsAfterNode(const Key& key, Node* n) const {
  // null n is considered infinite
  return (n != nullptr) && (compare_(n->key, key) < 0);
}
~~~
当要比较的对象（比如说是next节点指向的某一层对象）不为空且compare比较器得到的结果小于0时，说明这个key在顺序上是在n后面的，

接下来就是查找在每一层上
~~~
template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node*
SkipList<Key, Comparator>::FindGreaterOrEqual(const Key& key,
                                              Node** prev) const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    /* 获取当前 level 层的下一个节点 */
    Node* next = x->Next(level);

    if (KeyIsAfterNode(key, next)) {
      // Keep searching in this list
      x = next;
    } else {
      // prev 数组主要记录的就是每一层的 prev 节点，主要用于插入和删除时使用
      if (prev != nullptr) prev[level] = x;
      if (level == 0) {
        return next;
      } else {
        // Switch to next list
        level--;
      }
    }
  }
}
~~~
其中的GetMaxHeight函数获取的是当前结点的层数，我们从这个节点的最高层开始找，不断获取他的next节点，判断这个node他



