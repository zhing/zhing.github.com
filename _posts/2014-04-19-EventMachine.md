---
layout: post
title: "详解EventMachine的事件驱动模型"
description: ""
category: ruby
tags: [ruby]
---
我们在做ruby和java开发的时候，都感觉ruby开发要简介和方便不少，最明显的感触是遇到java并发的
时候，我们往往不知所措，因为好的并发程序实在很难写，而且测试的时候缺乏并发场景，难度就更大
了。撇开语法不说，关键是因为ruby是默认单线程的(虽然ruby里也有线程，但是ruby本身对线程的调度
是单线程的，就是同一时间只能有一个线程在执行)，即便是在IO编程这样的并发场景，ruby也能通过
基于事件驱动的EventMachine来很好的解决，下面我们就来说一下事件驱动编程。

####什么是事件驱动？
事件驱动最典型的应用场景就是GUI程序的设计，我们通过点击鼠标来触发事件，实际上我们是在按钮上
关联注册了点击触发事件的函数，当我们点击按钮的时候，被关联的函数就会被调用，这就是事件驱动
最通俗的解释。

我们将事件驱动引入一般的编程场景中，非事件驱动的普通做法是程序执行到相应位置，然后去执行函数，
等待函数的返回，接着执行函数后面的代码，但是如果这个函数去处理IO，那么很可能程序会在这里阻塞，
等待速度较慢的IO响应。但是事件驱动的做法是，将这个函数与某个事件(还是以IO为例)关联，当IO到来
的时候，事件被触发，函数被调用。说到这里你可能还是不太理解他们的区别，想一想这样的场景，一个
服务端与多个客户端进行socket通信，一般java程序的做法是开辟多个线程，然后每个线程去等待IO，很有
可能就是所有的线程都在等待，这样我们就必然涉及到多线程的并行控制，你是否有能力去做好他们的同步？
但是事件驱动编程只有一个主进程去处理这些事情，当有一个IO到来的时候，这个主进程处理这个IO没有任何
问题，但是当多个IO到来的时候，事件驱动程序会去一个一个的(或者协同)处理这些事件，当然这个顺序是基于事件
到来的时间顺序的。实际情况表明，在很多情况下事件驱动程序比多线程的模型效率更高，因为它省去了
多线程切换带来的代价。

####EventMachine
刚开始接触EventMachine的时候，觉得很是困惑，它是如何处理多个连接的并发的？这个问题困扰了我
很久，直到理解了事件驱动，才觉得EventMachine的主体思想还真是简单。

cloud foundry的组件大多是以EventMachine开头的，EventMachine就是最典型的基于事件驱动编程，它
有一个主循环去处理任务，也就是说这个主循环一旦阻塞，EventMachine将啥也干不了，所以永远也不要阻塞
主循环。那你也可能会说，如果我一次IO的任务很重，需要花费很多的CPU时间，那么EventMachine的处理
办法是将大任务放到后台去执行，所以到这里我终于明白了defer(推迟执行)的意思.试想我们使用EventMachine
去处理IO，每个任务都耗费主循环大量的时间，使主循环的时间片很长，那么别的IO事件长时间得不到响应
，那将是多少糟糕的事情。

上面的defer调用就是通过引用其它线程来帮助主循环来执行大任务的方法，有经验的人一眼就能看出来，
defer方法的缺陷，defer后面的block块在线程中执行，返回一个结果给主循环，这样主循环就能知道大任务
处理的结果，这样毕竟将任务转移给了别的线程，需要通过参数和返回值来控制，可能需要较大开销。如果
我们更愿意将任务放在当前线程中执行(这样免去了传参的烦恼等)，而我们又不愿意去阻塞主循环，next_tick
方法很好的解决了这个问题，它能将跟在后面的任务块递归地分配到主循环不同的循环周期去执行，具体
的调度交给EM就行了，说到这里主循环有个更专业的名字：Reactor(差点忘了)。

####事件驱动的优势与缺陷
其实事件驱动主要依赖于Reactor，它本身是一个单线程，所以事件驱动的缺点很明显就是无法向多线程
那样运用多核的优势，但是大多少场景下面事件驱动的性能是要优于多线程编程的，这得归功于不需要考虑
锁的因素，免去了线程切换的性能开销。

事件驱动的优势当然也很突出，就是对程序员的友好，让程序员免去了多线程编程的困扰。

当然我们也得选择好事件驱动编程的应用场景，现在很多高性能的IO框架，如java的Mina，netty等都无一例外
地采用了事件驱动的思想，从这里我们可能看出，事件驱动适合于IO密集的时候，而对于CPU密集(即CPU
繁忙的场景)则不太适合。

这个问题是我在尝试使用java去改写cloudfoundry的nats组件的时候困惑的。深有感触，在云平台中使用
ruby开发真比java省心不少。每天进步一点点，开兴~




     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
     
