---
layout: post
title: C++11原子操作与无锁编程
date: 2017-01-10
tags: [C++, 原子操作, 自旋锁, 无锁编程]
---

不讲语言特性，只从工程角度出发，个人觉得C++标准委员会在C++11中对多线程库的引入是有史以来做得最人道的一件事；今天我将就C++11多线程中的atomic原子操作展开讨论；比较互斥锁，自旋锁(spinlock)，无锁编程的异同，并进行性能测试；最后会讨论一下内存序的问题；为了流畅阅读你最好先熟悉一下C++11 Atomic的基本操作[英文文档](http://en.cppreference.com/w/cpp/atomic){:target="_blank"}，这里还有一份我觉得做得很用心的关于C++11并发编程的[中文教程](https://github.com/forhappy/Cplusplus-Concurrency-In-Practice){:target="_blank"}，你也可以从其中找到对应的知识点；

### 原子操作 ###

我们写的代码最终都会被翻译为CPU指令，一条最简单加减法语句都有可能会被翻译成几条指令执行；为了避免语句在CPU这一层级上的指令交叉带来的行为不可知，在多线程程序设计时我们必须通过一些方式来进行规范；这里面最常见的做法就是引入互斥锁，其大概的模型就是篮球模式：几个人一起抢球，谁抢到了谁玩，玩完了再把球丢出来重新抢；但互斥锁是操作系统这一层级的，最终映射到CPU上也是一堆指令，是指令就必然会带来额外的开销；

既然CPU指令是多线程不可再分的最小单元，那我们如果有办法将代码语句和指令对应起来，不就不需要引入互斥锁从而提高性能了吗? 而这个对应关系就是所谓的**原子操作**；在C++11的atomic中有两种做法:

> + 模拟, 比如说对于一个atomic\<T\>类型，我们可以给他附带一个mutex，操作时lock/unlock一下，这种在多线程下进行访问，必然会导致线程阻塞；
> + 有相应的CPU层级的对应，这就是一个标准的***lock-free***类型；
>
> *可以通过[is_lock_free](http://en.cppreference.com/w/cpp/atomic/atomic/is_lock_free){:target="_blank"}函数，判断一个atomic是否是lock-free类型*

### 自旋锁 ###

使用原子操作模拟互斥锁的行为就是自旋锁，互斥锁状态是由操作系统控制的，自旋锁的状态是程序员自己控制的；要搞清楚自旋锁我们首先要搞清楚自旋锁模型，常用的自旋锁模型有：

> 1. **TAS**, [<u>Test-and-set</u>](https://en.wikipedia.org/wiki/Test-and-set){:target="_blank"}，有且只有atomic_flag类型与之对应
> 2. **CAS**, [<u>Compare-and-swap</u>](https://en.wikipedia.org/wiki/Compare-and-swap){:target="_blank"}，对应atomic的compare\_exchange\_strong 和 compare\_exchange\_weak，这两个版本的区别是：Weak版本如果数据符合条件被修改，其也可能返回false，就好像不符合修改状态一致；而Strong版本不会有这个问题，但在某些平台上Strong版本比Weak版本慢 *[<u>注:在x86平台我没发现他们之间有任何性能差距</u>]*；绝大多数情况下，我们应该优先选择使用Strong版本；

我针对这两种模型分别实现了两个版本的自旋锁，最终代码可以在性能测试章节中找到，这里我们要注意以下问题：

> LOCK时自旋锁是自己轮询状态，如果不引入中断机制，会有大量计算资源浪费到轮询本身上；常用的做法是使用yield切换到其他线程执行，或直接使用sleep暂停当前线程.


### 无锁编程 ###

如果看了CAS实现的自旋锁代码会发现其有些别扭：每次都需要去重置exp的状态为false；CAS虽然也能实现自旋锁，但通常被我们用来进行无锁编程；
什么是无锁编程呢，让我们以一个例子开始:

> ```c++
> template<typename _Ty>
> struct LockFreeStackT
> {
>   struct Node
>   {
>     _Ty val;
>     Node* next;
>   };
>   LockFreeStackT() : head_(nullptr) {}
>   void push(const _Ty& val)
>   {
>     Node* node = new Node{ val, head_.load() };
>     while (!head_.compare_exchange_strong(node->next, node)) {
>     }
>   }
>   void pop()
>   {
>     Node* node = head_.load();
>     while (node && !head_.compare_exchange_strong(node, node->next)) {
>     }
>     if (node) delete node;
>   }
>   std::atomic<Node*> head_;
> };
> ```

整个逻辑很简单，pop只是push的逆过程，这里我们仅仅只分析一下push：每次push时去获得栈顶元素，生成一个指向栈顶的新元素，然后使用CAS操作去更新栈顶，这里可能有两种情况：

> 1. 如果新元素的next和栈顶一样，证明在你之前没人操作它，使用新元素替换栈顶退出即可；
> 2. 如果不一样，证明在你之前已经有人操作它，栈顶已发生改变，该函数会自动更新新元素的next值为改变后的栈顶；然后继续循环检测直到状态1成立退出；

不难看出，其实所谓无锁编程只是**将多条指令合并成了一条指令形成一个逻辑完备的最小单元，通过兼容CPU指令执行逻辑形成的一种多线程编程模型**；结束了吗，再等等，使用上面的代码，有很大的几率出delete不存在的内存，或内存被多次delete的错误，让我们进入下一节.

### ABA问题 ###

[维基百科: ABA problem](https://en.wikipedia.org/wiki/ABA_problem){:target="_blank"}，如果有两个线程[1&2]操作上面的堆栈，初始状态有2个元素: top->A->B，线程1执行pop操作，在CAS前进行线程切换:

> 注意pop函数中的head_.compare_exchange_strong(node, node->next)语句，如果对C/C++不够熟悉，很容易发生误解；我们不考虑函数包装等复杂情况，只考虑最简单的情况下在调用CAS原子操作前至少还有参数压栈操作，也就是说node->next不是调用时确定的，而是在参数压栈时确定的；前面说的CAS操作前进行线程切换，切换时{node, node->next}对应的是{A, B}.

然后线程2执行pop操作，将A，B都删掉，然后创建了一个新元素push进去，因为操作系统的内存分配机制会重复使用之前释放的内存，恰好push进去的内存地址和A一样，我们记为A'，这时候切换到线程1，CAS操作检查到A没变化成功将B设为栈顶，但B是一个已经被释放的内存块...

解决ABA问题的办法无非就是通过打标签标识A和A'为不同的指针，这下总结束了吧，事实上还没有，再次进入下一节.

### 内存回收 ###

还是先看上面的代码，在Pop时先获得了头指针*Node\* node = head_.load();*，如果这时候发生线程切换并且这个节点被另一个线程删除，当线程切回时候node无效造成node->next访问异常...，对于这个问题，现在流行的处理方式主要有：

> 1. Lock-Free Reference Counting: 引用计数方式，严格的说应该是全局计数方式；每次POP时首先增加计数，然后处理完后做检测，计数如果大于1证明其他线程也在操作，把节点存起来；只要检测到计数等于1，证明目前只有自己操作，可以删除当前节点和之前缓存的节点，删除之前节点时必须进行二次判断来解决交换后，其他线程Acquire的问题；具体可以参见Paper: [Lock-Free Reference Counting](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.92.8221&rep=rep1&type=pdf){:target="_blank"}, [Efficient and Reliable Lock-Free Memory Reclamation Based on Reference Counting](http://www.cse.chalmers.se/~tsigas/papers/MemoryReclamation-ReferenceCounting-ISPAN05.pdf){:target="_blank"}
> 2. Hazard pointer: 使用POP的线程，都有一个自己可以写的thread_local变量，每次得到的旧指针都存入其中，当交换完成后将该变量置为0；对于其他线程来说每次删除前对整个Hazard pointer表遍历一遍，如果发现其他线程也在访问，就把要删除的节点放入将删列表中，否则可以直接删除；定时对将删列表中的节点按照之前的规则和整个Hazard pointer表做比较，只要Hazard pointer表不持有这个节点就可删除；可参考[维基百科: Hazard pointer](https://en.wikipedia.org/wiki/Hazard_pointer){:target="_blank"}
> 3. Epoch Based Reclamation
> 4. Quiescent State Based Reclamation

我实现了前两种方式，后两种方式如果以后有时间再补上；

### 性能测试 ###

本来想挖个坑，在Github上重新建一个项目的；但工作太忙，给自己挖的坑也太多了，挖了估计也仅仅只是挖了，没精力去维护；好在代码是完整的，逻辑也相对比较简单，将以下代码拷到一个CPP文件中编译一下就能运行；测试上在VS2015中基本全覆盖的单元测试跑过，macOS上也经过了大概8个小时的压力测试，应该没太大问题，如果有问题请与我联系，先贴代码：

> ```c++
> #define _ENABLE_ATOMIC_ALIGNMENT_FIX 1    // VS2015 SP2 BUG
> #include <iostream>
> #include <thread>
> #include <mutex>
> #include <atomic>
> #include <chrono>
> #include <cassert>
> template<typename _Ty>
> struct NodeT
> {
>   std::unique_ptr<_Ty> data;
>   NodeT* next;
>   NodeT(const _Ty& _val, NodeT* _next)
>     : data(new _Ty(_val))
>     , next(_next)
>   {
>   }
> };
> template<size_t _SleepWhenAcquireFailedInMicroSeconds = size_t(-1)>
> class SpinLockByTasT
> {
>   std::atomic_flag locked_flag_ = ATOMIC_FLAG_INIT;
> public:
>   void lock()
>   {
>     while (locked_flag_.test_and_set()) {
>       if (_SleepWhenAcquireFailedInMicroSeconds == size_t(-1)) {
>         std::this_thread::yield();
>       } else if (_SleepWhenAcquireFailedInMicroSeconds != 0) {
>         std::this_thread::sleep_for(std::chrono::microseconds(_SleepWhenAcquireFailedInMicroSeconds));
>       }
>     }
>   }
>   void unlock()
>   {
>     locked_flag_.clear();
>   }
> };
> template<size_t _SleepWhenAcquireFailedInMicroSeconds = size_t(-1)>
> class SpinLockByCasT
> {
>   std::atomic<bool> locked_flag_ = ATOMIC_VAR_INIT(false);
> public:
>   void lock()
>   {
>     bool exp = false;
>     while (!locked_flag_.compare_exchange_strong(exp, true)) {
>       exp = false;
>       if (_SleepWhenAcquireFailedInMicroSeconds == size_t(-1)) {
>         std::this_thread::yield();
>       } else if (_SleepWhenAcquireFailedInMicroSeconds != 0) {
>         std::this_thread::sleep_for(std::chrono::microseconds(_SleepWhenAcquireFailedInMicroSeconds));
>       }
>     }
>   }
>   void unlock()
>   {
>     locked_flag_.store(false);
>   }
> };
> template<typename _Ty, typename _Lock>
> class LockedStackT
> {
>   typedef NodeT<_Ty> Node;
> public:
>   ~LockedStackT()
>   {
>     std::lock_guard<_Lock> lock(lock_);
>     while (head_) {
>       Node* node = head_;
>       head_ = node->next;
>       delete node;
>     }
>   }
>   void Push(const _Ty& val)
>   {
>     Node* node(new Node(val, nullptr)); // 不需要锁构造函数，这个可能是一个耗时操作
>     std::lock_guard<_Lock> lock(lock_);
>     node->next = head_;
>     head_ = node;
>   }
>   std::unique_ptr<_Ty> Pop()
>   {
>     std::unique_ptr<_Ty> ret;
>     Node* node;
>     {
>       // 同上，只需要锁链表本身，其他操作可以放到链表外执行
>       std::lock_guard<_Lock> lock(lock_);
>       node = head_;
>       if (node) head_ = node->next;
>     }
>     if (node) {
>       ret.swap(node->data);
>       delete node;
>     }
>     return std::move(ret);
>   }
>  
> private:
>   Node* head_ = nullptr;
>   _Lock lock_;
> };
> template<typename _Node>
> class MemoryReclamationByReferenceCountingT
> {
>   std::atomic<size_t> counter_ = ATOMIC_VAR_INIT(0);
>   std::atomic<_Node*> will_be_deleted_list_ = ATOMIC_VAR_INIT(nullptr);
>   void InsertToList(_Node* first, _Node* last)
>   {
>     last->next = will_be_deleted_list_;
>     while (!will_be_deleted_list_.compare_exchange_strong(last->next, first));
>   }
>   void InsertToList(_Node* head)
>   {
>     _Node* last = head;
>     while (_Node* next = last->next) last = next;
>     InsertToList(head, last);
>   }
> public:
>   ~MemoryReclamationByReferenceCountingT()
>   {
>     // 如果线程正常退出，Reference Counting算法能删除所有数据
>     assert(will_be_deleted_list_.load() == nullptr);
>     assert(counter_.load() == 0);
>     _Node* to_delete_list = will_be_deleted_list_.exchange(nullptr);
>     while (to_delete_list) {
>       _Node* node = to_delete_list;
>       to_delete_list = node->next;
>       delete node;
>     }
>   }
>   inline void Addref()
>   {
>     ++counter_;
>   }
>   inline bool Store(_Node* node) { return true; }
>   bool Release(_Node* node)
>   {
>     if (!node) return false;
>     if (counter_ == 1)
>     {
>       _Node* to_delete_list = will_be_deleted_list_.exchange(nullptr);
>       if (!--counter_) {
>         while (to_delete_list) {
>           _Node* node = to_delete_list;
>           to_delete_list = node->next;
>           delete node;
>         }
>       } else if (to_delete_list) {
>         InsertToList(to_delete_list);
>       }
>       delete node;
>     } else {
>       if (node) InsertToList(node, node);
>       --counter_;
>     }
>     return true;
>   }
> };
> template<class _Node, size_t _MaxPopThreadCount = 16>
> class MemoryReclamationByHazardPointerT
> {
>   struct HazardPointer
>   {
>     std::atomic<std::thread::id> id;
>     std::atomic<_Node*> ptr = ATOMIC_VAR_INIT(nullptr);
>   };
>   HazardPointer hps_[_MaxPopThreadCount];
>   std::atomic<_Node*> will_be_deleted_list_ = ATOMIC_VAR_INIT(nullptr);
>   void _ReleaseImpl(_Node* node)
>   {
>     // 检查HazardPointers中是否有线程正在访问当前指针
>     size_t i = 0;
>     while (i < _MaxPopThreadCount) {
>       if (hps_[i].ptr.load() == node)
>         break;
>       ++i;
>     }
>     if (i == _MaxPopThreadCount) { // 无任何线程正在访问当前指针，直接删除
>       delete node;
>     } else {  // 有线程正在访问，加入缓存列表
>       node->next = will_be_deleted_list_.load();
>       while (!will_be_deleted_list_.compare_exchange_strong(node->next, node));
>     }
>   }
> public:
>   ~MemoryReclamationByHazardPointerT()
>   {
>     // 自己不能删除自己，正常退出HazardPointer始终会持有一个节点，只能在此做清理
>     size_t count = 0;
>     _Node* to_delete_list = will_be_deleted_list_.exchange(nullptr);
>     while (to_delete_list) {
>       _Node* node = to_delete_list;
>       to_delete_list = node->next;
>       delete node;
>       ++count;
>     }
>     assert(count < 2);
>   }
>   inline void Addref() {}
>   bool Store(_Node* node)
>   {
>     struct HazardPointerOwner
>     {
>       HazardPointer* hp;
>       HazardPointerOwner(HazardPointer* hps)
>         : hp(nullptr)
>       {
>         for (size_t i = 0; i < _MaxPopThreadCount; ++i) {
>           std::thread::id id;
>           if (hps[i].id.compare_exchange_strong(id, std::this_thread::get_id())) {
>             hp = &hps[i];
>             break;
>           }
>         }
>       }
>       ~HazardPointerOwner()
>       {
>         if (hp) {
>           hp->ptr.store(nullptr);
>           hp->id.store(std::thread::id());
>         }
>       }
>     };
>     thread_local HazardPointerOwner owner(hps_);
>     if (!node || !owner.hp) return false;
>     owner.hp->ptr.store(node);
>     return true;
>   }
>   bool Release(_Node* node)
>   {
>     if (!node) return false;
>     _ReleaseImpl(node); // 对当前传入指针进行释放操作
>                         // 循环检测will_be_deleted_list_, 可以另开一个线程定时检测效率会更高
>     _Node* to_delete_list = will_be_deleted_list_.exchange(nullptr);
>     while (to_delete_list) {
>       _Node* node = to_delete_list;
>       to_delete_list = node->next;
>       _ReleaseImpl(node);
>     }
>     return true;
>   }
> };
> template<typename _Ty, typename _MemoryReclamation>
> class LockFreeStackT
> {
>   typedef NodeT<_Ty> Node;
>   struct TaggedPointer
>   {
>     Node* ptr;
>     size_t tag;
>     TaggedPointer() {}
>     TaggedPointer(Node* _ptr, size_t _tag)
>       : ptr(_ptr)
>       , tag(_tag)
>     {
>     }
>   };
> public:
>   ~LockFreeStackT()
>   {
>     TaggedPointer o(nullptr, 0);
>     head_.exchange(o);
>
>     Node* head = o.ptr;
>     while (head) {
>       Node* node = head;
>       head = node->next;
>       delete node;
>     }
>   }
>   void Push(const _Ty& val)
>   {
>     TaggedPointer o = head_.load();
>     TaggedPointer n(new Node(val, o.ptr), o.tag + 1);
>     while (!head_.compare_exchange_strong(o, n)) {
>       n.ptr->next = o.ptr;
>       n.tag = o.tag + 1;
>     }
>   }
>   std::unique_ptr<_Ty> Pop()
>   {
>     memory_reclamation_.Addref();
>     TaggedPointer o = head_.load(), n;
>     while (true) {
>       if (!o.ptr) break;
>       memory_reclamation_.Store(o.ptr);
>       // HazardPointer算法储存(相当于上锁)后，需要对有效值进行二次确认，否则还是有先删除的问题
>       // 这样做并没效率问题，不等的情况CAS操作也会进行循环，因此可以作为针对任何内存回收算法的固定写法
>       const TaggedPointer t = head_.load();
>       if (memcmp(&t, &o, sizeof(TaggedPointer))) {
>         o = t;
>         continue;
>       }
>       n.ptr = o.ptr->next;
>       n.tag = o.tag + 1;
>       if (head_.compare_exchange_strong(o, n)) break;
>     }
>     memory_reclamation_.Store(nullptr);
>     std::unique_ptr<_Ty> ret;
>     if (o.ptr) {
>       ret.swap(o.ptr->data);
>       memory_reclamation_.Release(o.ptr);
>     }
>     return std::move(ret);
>   }
> private:
>   std::atomic<TaggedPointer> head_ = ATOMIC_VAR_INIT(TaggedPointer(nullptr, 0));
>   _MemoryReclamation memory_reclamation_;
> };
> template<typename _Ty, int _ThreadCount = 16, int _LoopCount = 100000>
> struct LockFreePerformanceTestT
> {
>   template<class _ProcessUnit>
>   static double Run(_ProcessUnit puf)
>   {
>     std::thread ths[_ThreadCount];
>     auto st = std::chrono::high_resolution_clock::now();
>     for (int i = 0; i < _ThreadCount; ++i) ths[i] = std::thread([&puf]() {
>       for (int i = 0; i < _LoopCount; ++i) {
>         puf();
>       }
>     });
>     for (int i = 0; i < _ThreadCount; ++i) ths[i].join();
>     const double period_in_ms = static_cast<double>((std::chrono::high_resolution_clock::now() - st).count())
>       / std::chrono::high_resolution_clock::period::den * 1000;
>     return period_in_ms;
>   }
>   static void Run()
>   {
>     _Ty s;
>     std::cout << Run([&s]() {s.Push(0); }) << "\t\t";
>     std::cout << Run([&s]() { s.Pop(); }) << std::endl;
>   }
> };
> int main()
> {
>   std::cout << "LockedStack with std::mutex" << "\t\t\t\t\t";
>   LockFreePerformanceTestT<LockedStackT<uint32_t, std::mutex>>::Run();
>   std::cout << "LockedStack with SpinLockByTas yield" << "\t\t\t\t";
>   LockFreePerformanceTestT<LockedStackT<uint32_t, SpinLockByTasT<>>>::Run();
>   std::cout << "LockedStack with SpinLockByCas yield" << "\t\t\t\t";
>   LockFreePerformanceTestT<LockedStackT<uint32_t, SpinLockByCasT<>>>::Run();
>   std::cout << "LockedStack with SpinLockByTas usleep(5)" << "\t\t\t";
>   LockFreePerformanceTestT<LockedStackT<uint32_t, SpinLockByTasT<5>>>::Run();
>   std::cout << "LockedStack with SpinLockByCas usleep(5)" << "\t\t\t";
>   LockFreePerformanceTestT<LockedStackT<uint32_t, SpinLockByCasT<5>>>::Run();
>   std::cout << "LockFreeStack with MemoryReclamationByReferenceCounting" << "\t\t";
>   LockFreePerformanceTestT<LockFreeStackT<uint32_t, MemoryReclamationByReferenceCountingT<NodeT<uint32_t>>>>::Run();
>   std::cout << "LockFreeStack with MemoryReclamationByHazardPointer" << "\t\t";
>   LockFreePerformanceTestT<LockFreeStackT<uint32_t, MemoryReclamationByHazardPointerT<NodeT<uint32_t>>>>::Run();
>   return 0;
> }
> ```

测试结果如下：
![](/assets/cpp11_atomic_and_lockfree_program/proformance_test_excel.png)

> *注：在Windows x64下，atomic<size=16>不是lockfree，如果有需要要可以用InterlockedCompareExchange128自己实现一下；*

### 内存模型 ###

C++11原子操作的很多函数都有个std::memory_order参数，这个参数就是这里所说的内存模型，其并不是类似POD的内存布局，而是一种数据同步模型，准确说法应该是储存一致性模型，其作用是对同一时间的读写操作进行排序；C++11中一个定义了6种类型，我们可以将其分为4类，下面我从我的角度以普通程序员能理解的语言描述一下，具体的可以参见 [C++11 Memory Order](http://en.cppreference.com/w/cpp/atomic/memory_order){:target="_blank"}：

> 1. memory_order_relaxed: 很多文档都说这种模型是完全乱序的，但我理解同一线程内，基本上应该还是按照代码顺序执行的；
> 2. memory_order_release & memory_order_acquire: 两个线程A&B，A线程Release后，B线程Acquire能保证一定读到的是最新被修改过的值；这种模型更强大的地方在于它能保证发生在A-Release前的所有写操作，在B-Acquire后都能读到最新值；
> 3. memory_order_release & memory_order_consume: 上一个模型的同步是针对所有对象的，这种模型只针对依赖于该操作涉及的对象：比如这个操作发生在变量a上，而s = a + b; 那s依赖于a，但b不依赖于a; 当然这里也有循环依赖的问题，例如：t = s + 1，因为s依赖于a，那t其实也是依赖于a的；
> 4. memory_order_seq_cst: 顺序一致性模型，这是C++11原子操作的**默认模型**；大概行为为对每一个变量都进行2中所说的Release-Acquire操作，当然这也是一个最慢的同步模型；

说到内存模型，就不得不提一下经常被大家误用的 ***volatile*** 关键字，这个关键字仅仅保证：**数据只在内存中读写**，直接操作它既不能保证操作是atomic的，也不能保证Memory Order；其实在我理解中，这个应该是嵌入式，内核或驱动程序员专用关键字:)，当然如果在竞争不敏感的环境中用来做flag用一下也没太大问题.

最后要说一下x86体系中Release-Acquire是自动获取的，最终形成一个***memory_order_seq_cst***模型；因此绝大多数情况下***memory_order_relaxed***其实并没有什么用.

### 结语 ###

无锁编程真的很难，如果要完全写对那就成了变态难，出错了平时常见的调试手段根本没用，几乎全靠脑补；并且这种轮询方式相对于锁的中断挂起方式来讲，只有在超高并发的前提下才能达到一个理想的效果，低并发下空载会对系统资源造成极大的浪费，因此原则上我不推崇这个玩意；从测试结果来看，所有平台上自旋锁性能都非常接近无锁实现，并且其使用方式和互斥锁几乎没差别，因此在没啃透之前，使用锁的方式才是明智的选择.

### 参考资料 ###

[C++ 并发编程指南](https://github.com/forhappy/Cplusplus-Concurrency-In-Practice){:target="_blank"}  
[Yet another implementation of a lock-free circular array queue](https://www.codeproject.com/articles/153898/yet-another-implementation-of-a-lock-free-circular){:target="_blank"}  
[Common Pitfalls in Writing Lock-Free Algorithms](http://blog.memsql.com/common-pitfalls-in-writing-lock-free-algorithms/){:target="_blank"}  
[Writing Lock-Free Code: A Corrected Queue](http://www.drdobbs.com/parallel/writing-lock-free-code-a-corrected-queue/210604448){:target="_blank"}  
[维基百科: ABA problem](https://en.wikipedia.org/wiki/ABA_problem){:target="_blank"}  
[Lock-Free Reference Counting](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.92.8221&rep=rep1&type=pdf){:target="_blank"}  
[Efficient and Reliable Lock-Free Memory Reclamation Based on Reference Counting](http://www.cse.chalmers.se/~tsigas/papers/MemoryReclamation-ReferenceCounting-ISPAN05.pdf){:target="_blank"}  
[维基百科: Hazard pointer](https://en.wikipedia.org/wiki/Hazard_pointer){:target="_blank"}  
[C++11 Memory Order](http://en.cppreference.com/w/cpp/atomic/memory_order){:target="_blank"}  
[Nine ways to break your systems code using volatile](http://blog.regehr.org/archives/28){:target="_blank"}

<br/>

> [原始链接]({{page.url}}) 版权声明：自由转载-非商用-非衍生-保持署名 \| [Creative Commons BY-NC-ND 4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh)
