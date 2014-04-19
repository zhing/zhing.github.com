---
layout: post
title: "java并发之底层原理"
description: ""
category: java
tags: [java]
---
前面学习了java并发中实用的这么多方法和容器实现等，他们都离不开JVM的底层机制，它是如何实现synchronized、
volatile等一系列并发关键字的，相信对java感兴趣的朋友都不会放过这一节，也可以这么说如果不了解底层机制，
在做java性能优化的时候会处处掣肘。下面来一一介绍：

####volatile

  * 何为volatile？
  
     简单的说volatile就是轻量级的synchronized，它在多处理器开发中保证了共享变量的“可见性”，所谓的“可见性”
就是当一个线程修改一个共享变量的时候，另外一个线程能够读到这个修改的值。要完全理解清楚volatile，我们不得不
介绍一下处理器的缓存机制：

     * 缓冲行 缓存中可以分配的最小的存储单位。处理器填写缓存行的时候会加载整个缓存行，缓存行的大小依处理
    器的不同而异，一般是64个字节，也可能是32个字节。
    
     * 缓存行填充 当处理器识别到从内存中读取操作数是可缓存的，处理器读取整个缓存行到适当的缓存(L1,L2,L3...)
     
     * 缓存命中 如果进行高速缓存行填充操作的内存位置仍然是下次处理器访问的地址时，处理器从缓存中读取
     操作数，而不是内存。这里注意高速缓存行与缓存行的级联差别。
     
     * 写命中 当处理器将操作数写回到一个内存缓存的区域时，它首先会检查这个缓存的内存地址是否在缓存行中
     如果存在一个有效的缓存行，处理器将这个操作数写回缓存，而不是写回到内存。
     
  正因为如上文中缓存的存在，所以java线程对共享变量的写回操作不能及时的写回到内存中，其他线程对该共享变量
的访问操作可能都不到最新的值。所以volatile关键字能保证共享变量的及时写回，这就解释了所谓的”可见性“。

  * 为什么要使用volatile？
  
     在某些特定的场景下，为什么首选volatile，而不是synchronized?答案很简单，volatile的使用和执行成本更低，
因为它不会引起线程的上下文切换和调度。其实volatile关键字的使用场景很有限，所以要恰当。

  * volatile到底如何实现？
  
     我们先来看看JIT编译器会对java代码做什么：

         java代码：valatile Singleton instance = new Singleton();
         汇编代码：0x0la3deld: movb $0*0,0*1104800(%esi);0x0la3de24: LOCK add1 $0*0,(%esp)
     
   上面第二行汇编代码是一条带lock前缀的指令，它会引发两件事情：
     
     1. 将当前处理器缓存行的数据写回到系统内存。
     
     2. 这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效。
     
   那么lock指令是如何完成上面两个任务的呢？
   
     1. 在处理器写回内存的操作中，只能有一个处理器进行内存的写，有两种方式实现：一是总线锁定，导致其他
     cpu不能访问总线，也就意味着不能访问系统内存；二是锁定缓存，缓存一致性机制会阻止同时修改被两个以上
     处理器缓存的内存区域数据。显然锁缓存的代价更小。
     
     2. 那么缓存一致性是如何工作的呢，它因处理器而异，如IA-32和Intel64处理器能嗅探到其他处理器访问系统
     内存和他们的内部缓存。他们使用嗅探技术保证它的内部缓存，系统内存和其他处理器的缓存数据在总线上保持
     一致。而Pentinum和P6 family处理器中，通过嗅探一个处理器来检测其他处理器打算写内存，而这个地址当前
     处理共享状态，那么正在嗅探的处理器将无效它的缓存行，在下次访问相同内存地址时，强制执行缓存填充。
     
####原子操作
     
  我们先来看一组代码：
  
         import java.util.ArrayList;
         import java.util.List;
         import java.util.Map;
         import java.util.Queue;
         import java.util.concurrent.*;
         import java.util.concurrent.atomic.AtomicInteger;

         public class Counter {
      
          private AtomicInteger atomicI = new AtomicInteger(0);
          private volatile int i=0;
          public static void main(String[] argv){
            final Counter cas = new Counter();
            List<Thread> ts = new ArrayList<Thread>();
            long start = System.currentTimeMillis();
          
          for(int j=0;j<100;j++){
              Thread t = new Thread(new Runnable(){
        		  public void run(){
        			  for(int i=0;i<10000;i++){
        				  cas.count();
        				  cas.safeCount();
        			  }
        		  }
        	  });
        	  ts.add(t);
          }
          for(Thread t : ts){
        	  t.start();
          }
          for(Thread t : ts){
        	  try{
        		  t.join();
        	  }catch (InterruptedException e){
        		  e.printStackTrace();
        	  }
          }
          System.out.println(cas.i);
          System.out.println(cas.atomicI.get());
          System.out.print(System.currentTimeMillis()-start);
      } 
      
      private void safeCount(){
    	  
    	  for(;;){
    		  int i = atomicI.get();
    		  boolean suc = atomicI.compareAndSet(i, ++i);
    		  if(suc){
    			  break;
    		  }
    	  }
    	  //atomicI.addAndGet(1);
    	  
      }
      private void count(){
    	  i++;
      }
     }
   
   result：
           
     980238
     1000000
     61

  上面一段程序中显然i++，不是原子操作。在一个线程进行i++操作的时候，另外一个线程读取的可能是某个中间
状态，这样1自增两次结果是2的例子就不奇怪了。上面的代码中AtomicInteger是通过CAS(CPU硬件元语)的方式来实
现原子操作。

  * volatile可否实现上述原子操作？
  
     如果对volatile原理不清的人，首先会想上面的代码中如果将i定义为volatile类型的，是不是也能得到正确的
结果呢？答案显然是否定的。

     volatile只能保证可见性，并不能保证原子性。这就是为什么一个类的get方法可以使用volatile关键字，而
     set()方法则只能使用锁来保证一致性。有些人或许会问，get方法也没有必要使用volatile，答案很简单：你
     读取的数据可能在你的缓存里，而得不到新值。所以volatile适合于状态标志(true or false)的场景。
     
  那么有没有一种方法来既能保证原子性和又能保证可见性呢？
  
####synchronized关键字

     synchronized就是我们常见的和熟悉的“锁”。我们习惯于叫它重量级锁，但是JavaSE1.6对其进行改进以后，
    它变得没那么重量级了，这都是java1.6引入偏向锁和轻量级锁的贡献。
    
   * 锁怎么用？
   
     java的每一个对象都可以作为锁。

     * 对于同步方法，锁是当前的实例对象
     
     * 对于静态同步方法，锁是当前类的Class对象
     
     * 对于同步方法块，锁是synchronized方法里配置的对象
     
   * 锁怎么工作？
   
     JVM是基于进入和退出Monitor对象来实现方法同步和代码块同步。但是两者的实现细节不一样，我们来看一下
代码块同步。代码块同步是使用monitorenter和monitorexit指令实现的，monitorentor指令在编译后擦入到同步代码块的
开始位置，而monitorexit是插入到方法结束处和异常处，JVM要保证每个monitorentor必须有对应的monitorexit
与之配对。任何对象都有一个monitor与之关联，当且一个monitor被持有后，它将处于锁定状态。线程执行到monitorentor
指令时，将会尝试获取对象所对应的monitor的所有权，即尝试获得对象的锁。
     
     锁存在于java的对象头中，如果对象是数组类型，则java用三个字宽存储对象头，如果对象不是数组类型，则
使用两个字宽存储对象头。在32位机器中，一字宽等于4字节，64位机器中一字宽等于8字节。

   * 锁有哪些种类？
   
     1. 偏向锁

         Hotspot的作者经过研究发现大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得，所以
         为了让线程获得锁的代价更低引入了偏向锁。当一个线程访问同步块并获取锁的时候，会在对象头和栈帧
         中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，
         而只需简单的测试一下对象头里是否存储着指向当前线程的偏向锁。
         
     2. 轻量级锁
     
         线程在执行同步块之前，JVM会先在当前线程的栈帧中创建用于存储锁记录的空间，并将对象头中的Mark
         Word复制到锁记录中。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功
         ，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。
         
     3. 重量级锁
     
         重量级锁会使产生竞争的线程阻塞，等待锁被释放以后，重新加入竞争。
         
     对于上面三种锁，JVM会根据竞争的激烈程度对锁进行升级，但是不能降级，目的是为了提高获得锁和释放锁
     的效率。偏向锁的执行速度非常快，但是如果存在竞争，会带来额外的锁撤销的消耗，适合于只有一个线程
     访问同步快场景。轻量级锁不会使得线程阻塞，但是始终得不到锁竞争的线程使用自旋会消耗CPU，实用于追求
     响应时间的情形，其同步块的执行速度非常快。重量级锁会阻塞，也就是会发生线程上下文的切换和调度，
     但是不会消耗cpu，适合于追求吞吐量的场景。
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     


































    
    
    

    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    













  






































   
   
   
   
   
   
   
   
   
   
   
   
















        

   

     


















        























































        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        
        


































































  






























   
   
  
  
	
	
	
	
	
	
	
	
	
	
	
	
  