---
layout: post
title: "java并发之阻塞队列"
description: ""
category: java
tags: [java]
---
在生产者和消费者模型中，我们会使用到阻塞队列。java给我们提供了七种阻塞队列，基本上能满足各种使用
场景了。下面我们来分析一下：

####阻塞队列

  * 何为阻塞队列？
  
    一看到这个问题，很可能会想起来ConcurrentLinkedQueue是一种非阻塞的实现方式，阻塞队列具有
如下两个性质：

     * 支持阻塞的插入方法，就是在队列满的时候，队列会阻塞插入元素的线程，直到队列不满
     
     * 支持阻塞的移除方法，就是在队列为空的时候，队列会阻塞获取元素的线程，直到队列非空
     
   当阻塞队列不可用时，它提供了四种处理方式：
   
   1. 抛出异常：add(e)方法会在队列已满的时候，抛出异常IllegalStateException("Queue full"),使用
remove方法在队列为空的时候抛出NoSuchElementException异常。

   2. 返回特殊值：offer(e)方法往队列中插入元素的时候，会返回是否插入成功，poll方法从队列中拿
元素时，如果没有则返回null。

   3. 一直阻塞：调用put(e)方法，当队列已满的时候，队列会阻塞生产者线程，调用take()方法，当队列
为空的时候，会阻塞线程直到队列不为空。

   4. 超时退出：在上述第二种情况中，offer和poll方法分别传入时间参数，队列会阻塞线程一段时间，
如果超时则退出线程。

   现在我们终于对队列的这些方法有了个明确的认识，下面就看看java中提供了那些BlockingQueue吧。
   
* Java中有哪些阻塞队列？
 
   Java中一共提供了7种阻塞队列，分析如下：

   1. ArrayBlockingQueue，一个由数组结构构成的有界阻塞队列。默认的情况下不保证公平的访问队列，
也就是说最先阻塞的线程可能最后才访问队列，我们可以通过设置参数来保证队列的公平性，但是这样会
影响队列的吞吐量。

   2. LinkedBolckingQueue，一个用链表实现的有界阻塞队列。线程池中Executor.newFixedThreadPool
使用的就是这个队列。

   3. PriorityBlockingQueue，一个支持优先级的无界阻塞队列，默认情况下采取自然升序排列，也可以
通过compareTo方法来指定元素的排序规则。但是通优先级的情况下并不能保证顺序。

   4. DelayQueue，一个支持延时获取元素的无界阻塞队列，队列使用PriorityQueue来实现，队列中的元
素必须实现Delayed接口。创建元素的时候可以指定多久才能从队列中获取元素，只有在元素的延时期满的
时候才能从队列中获取它。网上说这个队列非常有用，还有待场景的剖析···

   5. SynchronousQueue，一个不存储元素的阻塞队列，每一个put操作必须等待一个take操作，否则不能
继续添加元素。如上文所述，它也可以构造其公平访问的策略，它就相当于一个数据的传递者，将数据从
生产者处传递给消费者，并且它的效率要高于ArrayBlockingQueue和LinkedBlockQueue。

  6. LinkedTransferQueue，是一个由链表结构组成的无界阻塞TransferQueue队列。

  7. LinkedBlockingDeque，是一个由链表结构组成的双向阻塞队列，所谓双向队列你可以从队列的两端
移出或者插入元素。

* 阻塞队列如何实现？

   java中的阻塞队列只用通知模式实现，所谓通知模式，就是当往一个满队列中添加元素的时候，生产者
阻塞，当有消费者从队列中拿走一个元素的时候，会通知生产者，队列可用。至于如何实现对一个线程的
阻塞和唤醒，则需要去看JVM底层的源码了。

####Fork/Join框架

* 何为Fork/Join框架？

  Fork/Join是Java7提供的一个用于执行并行任务的框架，简单地说Fork就是将一个大的任务切分成诺干个
小任务并行执行，Join就是将这些小任务的结果合并起来，得到大任务的结果。

  这里有一个很有趣的算法就是工作窃取算法，当我们将一个比较大的任务分割成若干互不干扰的任务的
时候，有的线程会先把任务干完，而有的线程的任务队列里还有任务待处理，这时候干完活的线程会等待
还没做完任务的线程返回，与其让它等着，不如让它去帮助干活。这么时候它们会访问同一个队列，可能
会产生竞争，为了减少竞争，可以使用双端队列。

  我们了解了fork/join框架的使用场景，明确了使用它的两个步骤：
  
     * 分割任务：我们需要一个fork类来将大任务分割成子任务。
     
     * 执行任务并合并结果，分割的子任务分别放在双端队列里，然后启动几个线程对双端队列中的任务执行，
并将执行完的结果放入另一个队列中，启动一个线程从队列中拿数据直到完成最后的结果。

* 如何使用Fork/Join框架？

      import java.util.concurrent.ExecutionException;
      import java.util.concurrent.ForkJoinPool;
      import java.util.concurrent.Future;
      import java.util.concurrent.RecursiveTask;
      public class CountTask extends RecursiveTask<Integer>{
      private static final int THRESHOLD =2;
      private int start;
      private int end;
      public CountTask(int start,int end){
          this.start = start;
    	  this.end = end;
      }
      
      protected Integer compute(){
    	  int sum=0;
    	  boolean canCompute=(end-start)<=THRESHOLD;
    	  
    	  if(canCompute){
    		  for(int i=start;i<=end;i++){
    			  sum+=i;
    		  }
    	  }else{
    		  
    		  int middle = (start+end)/2;
    		  CountTask leftTask=new CountTask(start,middle);
    		  CountTask rightTask = new CountTask(middle+1,end);
    		  
    		  leftTask.fork();
    		  rightTask.fork();
    		  
    		  int leftResult=leftTask.join();
    		  int rightResult=rightTask.join();
    		  
    		  sum = leftResult + rightResult;
    	  }
    	  return sum;
      }
      
      public static void main(String[] args){
    	  ForkJoinPool forkJoinPool = new ForkJoinPool();
    	  CountTask task = new CountTask(1,4);
    	  Future<Integer> result = forkJoinPool.submit(task);
    	  try{
    		  System.out.println(result.get());
    	  }catch(InterruptedException e){
    		  
    	  }catch(ExecutionException e){
    		  
    	  }
       }
      }

上述代码用Fork/Join框架演绎了1+2+3+4的任务分解与合并。

* Fork/Join框架如何实现？

   ForkJoinPool由ForkJoinTask数组和ForkJoinWorkerThread数组组成，前者负责存放程序提交给ForkJoinPool
的任务，后者主要负责执行任务。大致的工作流程如下：

   1. ForkJoinTask的fork方法会调用ForkJoinWorkerThread的pushTask方法，pushTask方法将当前任务放在ForkJoinTask数组queue里。

   2. 然后调用ForkJoinPool的signalWork()方法唤醒或是创建一个工作线程来执行任务。

   3. ForkJoinTask的join方法主要是阻塞当前线程并等待获取结果。

上面的过程已经相当清晰了

以上学习了java的各种并发容器和框架，不仅是学习层面上，在以后开发过程中也要好好体会。







































     
     
     
