# 底层实现

Monitor底层是由C++实现的，在C++代码中，Monitor其实就是一个C++的对象：`ObjectMonitor`。每个Java对象实例都有一个对应的Monitor对象，Monitor对象和Java对象一起被创建和销毁。源码可以参见openjdk。 [objectMonitor.hpp](http://hg.openjdk.java.net/jdk8u/jdk8u/hotspot/file/e2ac513ec7b3/src/share/vm/runtime/objectMonitor.hpp)

```cpp
  ObjectMonitor() {
    _header       = NULL;
    _count        = 0;
    _waiters      = 0, 
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; 
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ;
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
    _previous_owner_tid = 0;
  }
```

# 数据结构

- ObjectMonitor整体上可以分为两部分。
  - 一部分是是这个监控对象的基本信息表示当前锁的实时状态
  - 一部分表示各种情况下需要获取锁的排队信息
    ![](https://img-blog.csdnimg.cn/7866f3bfecf84fed8b7af0b3761a50e6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAc2t5X2NjeQ==,size_20,color_FFFFFF,t_70,g_se,x_16)

## 基本信息

- header: 用来保存锁对象的mark word的值。**因为object里面已经不保存mark word的原来的值了，保存的是ObjectMonitor对象的地址信息。** 当所有线程都完成了之后，需要销毁掉ObjectMonitor的时候需要将原有的header里面的值重新复制到mark word中来。
- object: 指向的是对象的地址信息，方便通过ObjectMonitor来访问对应的锁对象。
- owner: 指向的是当前获得线程的地址，用来判断当前锁是被哪个线程持有。

## 排队信息

- cxq: 线程刚进来的时候没有获取到锁的时候，在当前的排队队列。
- waitSet: 是指已经获取得一次锁了，对象调用了wait方法，讲当前线程挂起了就进入了等待队列。等待时间到期的时候唤醒，或者其他线程唤醒。
- entryList: 是队列用来获取锁的缓冲区，用来将cxq和waitSet中的数据 移动到entryList进行排队。这个统一获取锁的入口。一般是cxq或者waitSet数据复制过来进行统一排队。

# 实现

## exit

线程执行同步方法或代码块结束之后，会调用exit方法将当前线程从锁对象中移除，并尝试唤醒启动的线程来获取锁的过程。核心要点有两个:

- 一个是将当前线程释放，这个很好理解就把_owner字段中线程值设为空就好了。
- 另一个是唤醒其他线程，这个就涉及到相关的策略，因为前面他有一个排队队列_cxq，还有一个_EntryList，到底从哪里优先去取。提供了四种策略：
  - QMode=2 的时候 优先从_cxq唤醒
  - QMode=3 的时候 _cxq 中的数据加入到 _EntryList 尾部中来 然后从 _EntryList 开始获取
  - QMode=4的时候 _cxq 中的数据加入到 _EntryList 前面来 然后从 _EntryList 开始获取
  - QMode=1 的时候这个是**默认策略**优先从 _EntryList 中获取 如果 _EntryList 为空的情况，把 _cxq 进行队列反转然后从_cxq获取

    
