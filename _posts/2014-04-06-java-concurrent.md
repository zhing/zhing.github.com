---
layout: post
title: "java并发之容器分析"
description: ""
category: java
tags: [java]
---
谈到java并发，作为java中使用最为频繁的工具-容器，就得不说它的线程安全。在不考虑并发的情况下，我们使用
的Collections和Map都不是线程安全的，但是java中也有原生的线程安全的容器HashTable，但是它的效率比较低下，
所以java又增加了concurrent这个包，其中的很多包的实现值得我们来学习，下面就从ConcurrentHashMap和Concurrent
LinkedQueue两种使用最多的线程安全容器来分析。这方面的学习主要来自于方腾飞的《java并发编程的艺术》，特
推荐之。

####ConcurrentHashMap

   * 为什么不选择HashTable？
   
     罪魁祸首当然是效率底下。HashTable是使用synchronized来保证线程安全的，在线程激烈的情况下效率表现的
计较低下，因为当一个线程在访问HashTable的时候，其他线程去访问它的同步方法的时候，很可能处于轮询或者是
阻塞状态，当竞争越是激烈的时候，效率就越是低下。本质来说，使用synchronized来保证同步的时候，一个线程
使用get的操作的时候，其他的线程由于获取不了锁，所以就不能进行put及get等操作。那么ConcurrentHashMap是
怎么来避免对所有同步方法使用单一锁的呢?答案是分段锁。

   * 何为分段锁？
     
     分段锁既是对HashMap里的元素进行分段，然后对分段元素分别上锁，这样在访问不同段的元素的时候，只需要获得
本段锁即可，而无需去把整个HashMap锁住。这是ConcurrentHashMap分段锁的基本原理。
     
     ConcurrentHashMap是由Segment和HashEntry两个数组结构构成的，Segment是一种可重入锁结构，HashEntry
即是存储键值对的结构。它的分层结构表现如下：一个ConcurrentHaspMap包括一个Segment结构数组，而一个Segment
中又包含一个HashEntry结构数组，每个Segment结构管理着它里面的HashEntry结构数组的同步。那大部分人又有一个
疑问就是这相当于在Map和HashEntry中间加上了一层，如果不能保证Segment和HashEntry的均匀分布，那Concurrent
HashMap将会失去意义。那么ConcurrentHashMap又是如何保证元素结构的均匀分布的呢？

   * Segment的初始化
   
     说到ConcurrentHashMap的Hash分布，得先从其初始化说起，ConcurrentHashMap的初始化是由initialCapacity，
loadFactor，concurrencyLevel来完成的。initialCapacity是HashMap的容量，loadFactor是每个Segment的负载因子，
concurrencyLevel是HashMap中含有的Segment数组大小，它们都有一个默认值来初始化ConcurrentHashMap。

  Segment数组的大小用ssize来表示，ssize的大小是通过concurrentLevel来决定的，为了能够使用按位与的方式
来定位Segment，就必须保证ssize的大小是2^N,也就是说如果concurrentLvel的大小是14，15或者16，那么ssize就
等于16。concurrencyLevel的最大是65535，即Segment的最大长度是65535，对应的二进制为16位。部分源代码如下：

         if (concurrencyLevel > MAX_SEGMENTS)
            concurrencyLevel = MAX_SEGMENTS;
         // Find power-of-two sizes best matching arguments
         int sshift = 0;
         int ssize = 1;
         while (ssize < concurrencyLevel) {
            ++sshift;
            ssize <<= 1;
         }
         this.segmentShift = 32 - sshift;
         this.segmentMask = ssize - 1;
         Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    
   segmentMask就是段掩码，segmentShift既是段偏移。那么初始化每一个Segment又是怎么回事呢？
   
         if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
         int c = initialCapacity / ssize;
         if (c * ssize < initialCapacity)
            ++c;
         int cap = MIN_SEGMENT_TABLE_CAPACITY;
         while (cap < c)
            cap <<= 1;
         for(int i=0;i<this.segment.length();i++)
            this.segment[i]=new Segment<K,V>(cap,loader);
            
   由源码中可知，每个Segment的容量是通过均分的，里面的HashEntry数组的理论长度也是2^N，当然实际长度还
得乘以一个负载因子即Segment内HashEntry数组的容量为：threshold=cap*loader。当然其长度为2^N,当然也是为
了在Segment中通过按位与的方式迅速定位到HashEntry。

   * Segment如何定位？
   
     既然ConcurrentHashMap既然是通过每个Segment锁来保护各段的数据的，那么在插入和获取元素的时候是如何
快速定位到各个Segment的呢。我们知道Object类会为每个对象生成一个HashCode，它是对象的地址码映射，而Concurrent
HashMap会对其再进行一次hash，其目的当然是为了减少Hash冲突。试想一下如果我们不进行再hash，那么散列效果
将不尽如人意，假设我们只通过HashCode & SegmentMask来定位HashEntry所在的Segment，由于SegmentMask的默认
值是15，那么所有低15位地址相同的对象将被定位到同一个Segment中去，那么散列的效果将很差（按照程序局部
性原理，两个竞争者总是同在一个Segment将会使ConcurrentHashMap失去意义）。所以通过再次hash，使得对象地址
的每一位都参入到hash中去，散列的效果将大大提高。

* ConcurrentHashMap如何进行Get操作？
  
     Segment的Get操作实现非常地简洁高效，因为它根本不需要加锁。那么它是如何做到不加锁的呢，道理很简单
就是Get方法中的每个共享变量都会被定义为volatile变量，如统计当前Segment大小的count变量和存储值得HashEntry
的value，既然都被定义成为了volatile变量，那么就可以在多线程中保持可见性。由于对于volatile变量的写入
先于读操作，所以即使两个线程同时对一个volatile变量进行读写操作，也能保证Get操作得到的是最新值。
   
* Segment如何进行Put操作？
  
     为了线程安全，对变量进行put操作必须对共享变量进行加锁。put操作首先定位到Segment中去，然后在Segment
中进行插入操作。这里先会对Segment进行扩容，然后定位元素插入HashEntry数组的位置，写入即可。这里可以说的
是在判断是否需要扩容时比HashMap更合适，因为HashMap是在插入完成后判断是否需要扩容，而很大可能是上一次
扩容后没有新的元素插入，这就造成了一次无效的扩容，所以ConcurrentHashMap将检查提前是在有需要的时候扩容，
显然更加合理。

   那么如何进行扩容呢，这里是创建一个两倍于当前容量的数组，然后将原来数组中的元素插入到新数组中。当然
ConcurrentHashMap为了高效，不会对整个容器进行扩容，而只是对各Segment进行扩容。

* ConcurrentHashMap如何进行size操作
  
     前面已经说过Segment的全局变量count是一个volatile变量，那么是不是每次将所有的Segment的count相加都能
得到整个容器的size值呢?答案显然是否定的，因为在相加的过程中各个Segment的count值可能已经变化。当然一个
很简单的解决办法是在统计时，将各个Segment的改变count的方法全部锁住，这样做当然是低效的。

   一般的情况是统计过程中，各Segment的count值不会改变，所以在统计容器的size操作的时候，会查看各个Segment
的count是否改变，这就需要设置一个modCount变量，任何一个改变Segment内元素个数的方法都会使得modCount改变，
如果统计过程中modCount不改变，则统计成功，如果改变，则统计失败。如果尝试两次失败后，则换用加锁的方式
来对容器的size进行统计。

####ConcurrentLinkedQueue

队列也是java容器中很重要的组成部分，那么我们如何来使用一个线程安全的队列呢？线程安全的队列有两种实现
方式一个是阻塞式的队列，一个是非阻塞式的队列。阻塞式的队列很好理解就是使用一个锁(出队入队共用一个锁)
或是两把锁(出队入队分别加锁)来控制队列的同步，而非阻塞式就需要使用循环CAS的方式来实现，它的核心理念是
一个线程的挂起或是阻塞不应该影响另一个线程。下面我们来看ConcurrentLinkedQueue是如何实现非阻塞式队列的。

* 何为CAS操作？

   CAS操作最通俗的表达方式就是通过cpu的CAS指令来完成原子操作，几乎所有的原子的操作都是通过这种特性来
完成的，我们来看看java中的原子类的自增是如何保证原子性的。

         public final int incrementAndGet() {
         for (;;) {
           int current = get();
           int next = current + 1;
         if (compareAndSet(current, next))
            return next;
           }
         }

这里采用了CAS操作，每次从内存中读取数据，然后和+1后的数据进行CAS操作，如果成功则返回，如果失败则继续
尝试，compareAndSet方法使用JNI完成CPU指令的操作代码为：

         public final boolean compareAndSet(int expect, int update) {   
           return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
          }

这段代码我们应该很熟悉，在ConcurrentLinkedQueue源码中被使用到好几次。

* 如何保证入队同步？

   由于CAS循环的原理很清晰，我们直接进入源码：
   
        public boolean offer(E e) {
        checkNotNull(e);
        final Node<E> newNode = new Node<E>(e);//为对象创建一个新节点

        for (Node<E> t = tail, p = t;;) {
            Node<E> q = p.next; //获得tail节点的next节点
            if (q == null) {
                // 表明p为最后一个节点
                if (p.casNext(null, newNode)) {
                    if (p != t) // hop two nodes at a time
                        casTail(t, newNode);  // Failure is OK.
                    return true;
                }
                // Lost CAS race to another thread; re-read next
            }
            else if (p == q)
                // We have fallen off list.  If tail is unchanged, it
                // will also be off-list, in which case we need to
                // jump to head, from which all live nodes are always
                // reachable.  Else the new tail is a better bet.
                p = (t != (t = tail)) ? t : head;
            else
                // Check for tail updates after two hops.
                p = (p != t && t != (t = tail)) ? t : q;
          }
         }

   在源码中应该可以看出tail节点并不一定就是尾节点，入队的步骤分为两步，第一步就是定位尾节点，如果一个
节点的next节点为null，则其就是尾节点，然后把新节点插入到链表的尾部，这里会用到cas操作保证原子性，如果
失败继续循环。如果添加成功，会尝试将新节点设置成为tail节点，如果失败也会返回。这样就会解释两个问题，一
是为什么tail节点并不一定是尾节点，令一个是队列入队方法会始终返回true，所以判断一个入队操作是否成功是
徒劳的。

   这样也许会让你思考为什么不保证将新入队的节点设置为tail节点呢？这样是为了避免每次设置tail节点的CAS
循环带来的性能问题，也许你会接着问那么每次寻找尾节点就不会带来性能问题吗？或许这样做效果更好的原因是
更新tail节点需要的是volatile变量的写操作，而寻找尾节点是对volatile变量的读操作，显然读操作的代价更小。


   * 如何出队？
   
     借助我们对入队的理解，出队的原理就显得很简单了，源码如下：

         public E poll() {
         restartFromHead:
         for (;;) {
            for (Node<E> h = head, p = h, q;;) {
                E item = p.item;

                if (item != null && p.casItem(item, null)) {
                    // Successful CAS is the linearization point
                    // for item to be removed from this queue.
                    if (p != h) // hop two nodes at a time
                        updateHead(h, ((q = p.next) != null) ? q : p);
                    return item;
                }
                else if ((q = p.next) == null) {
                    updateHead(h, p);
                    return null;
                }
                else if (p == q)
                    continue restartFromHead;
                else
                    p = q;
            }
           }
         }

   首先获取队列的head节点元素，判断其是否为空，如果为空说明已经有一个线程将其出队了，如果不为空，则
使用CAS循环将其引用设置为null，如果成功则返回节点元素，如果不成功，则表明有线程将其出队了，需要重新获取
Head节点。

































    
    
    

    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    













  






































   
   
   
   
   
   
   
   
   
   
   
   
















        

   

     


















        























































        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        


































































  






























   
   
  
  
	
	
	
	
	
	
	
	
	
	
	
	
  