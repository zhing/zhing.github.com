---
layout: post
title: "java并发之线程池"
description: ""
category: java
tags: [java]
---
相信有很多java程序员在开发并发程序的时候会忽视线程池的作用，宁愿去用容器存储线程对象，可能是
觉得这样才对线程有完全的掌控力。这里就来聊一聊java的线程池ThreadPoolExecutor。

####线程池

   * 为什么需要线程池？
   
     合理使用线程池有三个好处：

    * 降低资源消耗，通过对线程的重复利用来降低线程的创建和销毁的消耗
    
    * 提高响应速度，任务到达时，任务可以不需要等到线程创建时就开始执行
    
    * 提高线程的可管理性，利用线程池可以进行统一的分配，调优和监控
    
 所以相比于使用HashMap来存储线程，使用线程池当然是首选。
 
   * 如何创建线程池？
  
         ThreadPoolExecutor(int corePoolSize,int maximumPoolSize,long keepAliveTime,
                              TimeUnit unit,BlockingQueue<Runnable> workQueue) 

     * corePoolSize：当提交一个任务到达线程池时会创建一个新线程，即使有其他的空闲基本线程存在，
     也会创建新线程，但是当线程池中的线程数达到corePoolSize的时候，就会停止创建新线程。
     
     * workQueue(任务队列)：常用的主要有两种任务队列，一是LinkedBlockingQueue，它是基于链表
     结构的阻塞队列，按照FIFO的方式排序元素，Executor.newFixedThreadPool()使用这个队列。二是
     SynchronousQueue，它是不存储元素的阻塞队列，每次插入操作必须等到另一个线程调用移除操作，它
     的吞吐量比第一种队列略高，Executor.newCachedThreadPool()使用的是这个队列。
     
     * maximumPoolSize：线程池允许的最大线程数。
    
     * keepAliveTime：线程池的工作线程空闲后，保持其存活的时间，如果任务很多，每个任务的执行
     时间比较短，可以加大这个时间，提高线程的利用率。
     
     * unit：线程活动的保存时间单位。
     
     这里我们需要解说一下corePoolSize与maxmumPoolSize的区别，其实可以简单地归纳为：当线程数少于
     corePoolSize时首选创建新线程来执行任务，而不排队；当线程池中的线程数等于或多于corePoolSize时
     首选排队而不创建新线程。那么当队列满的时候，情况又将如何呢？
     
     答案是：如果不能入队，则创建新的线程来执行任务，如果此时线程数超过maximumPoolSize，则任务
     将被拒绝。所以当队列是一个无界队列的时候，即队列不会到满时，maximumPoolSize将会失去意义。
     
* 如何向线程池提交任务？

     向线程池提交任务有两种方式，一种是直接提交：
     
         threadsPool.execute(new Runnable());
    
     这种方式无法判断任务被成功执行与否。还有一种方式是submit方法：
     
         Future<Object> future=executor.submit(ReturnValueTask){
            try{
                Object s= future.get();
            }
            ...异常处理
         }
    
     上面的get方法将会阻塞直到任务完成为止，通过这个方法可以获取线程的返回值。
     
* 如何关闭线程池？
  
     关闭线程池比较简单就是调用线程池的shutdown或者shutdownNow方法，他们的原理都是遍历线程池中
的所有线程，然后依次调用线程的interrupt方法来终止线程。我们通过上述两种方法的字面就可以看出两者
的差别，就是shutdownNow会将线程池的状态设置成stop，然后去尝试中断所有正在执行任务的线程，而
shutdown将线程池的状态设置成shutdown，然后去中断所有当前没在执行任务的线程。具体使用那种方式
得看具体任务。

* 线程池的工作流程如何？

     一个任务提交到线程池中大致分为这四个步骤，如下(其实上文中已经提到)：
     
     1. 如果核心工作线程未满(即线程数小于corePoolZize)，则创建新线程执行任务，否则判断2
     
     2. 如果队列未满，则将任务加入队列中排队等候，否则判断3
     
     3. 如果总的线程数为达到最大线程数(即maximumPoolSize)，则创建新的线程来执行任务，否则判断4
     
     4. 拒绝该任务
     
* 如何合理地配置线程池?

     根据任务的特性，可以从以下几个方面来考虑：
     
     1. 任务的性质：
        
        * cpu密集型：任务配置尽可能少的线程，如配置cpu+1个线程的线程池。
        
        * IO密集型：由于线程大部分时间都在IO等待，可以配置尽可能多的线程，如2*cpu；
        
        * 混合型：可以对线程进行拆分，如果运行时间比相差太大则没有必要进行拆分。
        
     2. 任务的优先级：
     
        优先级不同的任务可以使用，PriorityBlockingQueue来处理，让优先级高的先处理。
        
     3. 任务执行时间的长短：
     
        执行时间不同的任务可以交给规模不同的线程池来处理，也可以使用优先级来控制，即时间短
        的任务先处理。
        
     4. 任务的依赖性：
     
        最典型的例子就是有数据库连接的时候，因为线程提交SQL后需要等待数据库的返回，这时候线程
        数设置的越大，才能更好地使用cpu。
     
  最后一条建议就是尽量使用有界队列，如果使用无界队列，在任务密集的时候，任务被不断地提交到
线程池中，这时候如果发生故障，那么一来不容易尽早发现问题所在，二来内存被撑爆，可能影响到别的
业务。

*  线程池如何监控？

     通过线程池提供的参数可以对他进行监控：
     
     * taskCount：线程池需要执行的线程数量
     
     * completedTaskCount：已经完成的任务数量
     
     * largestPoolSize: 线程池曾经创建的最大线程数，如果等于maximumPoolSize，表示线程池曾经满过
     
     * getPoolSize：得到当前线程池中的线程数量，因为线程在线程池中不会销毁，所以这个数一般只增不减
     
     * getActiveCount：获取活动的线程数
     
  当然我们也可以通过继承线程池来重写其中的方法如beforeExecute，afterExecute或者terminated方法
在执行任务前，执行任务后和线程池关闭前干一些事情。
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
