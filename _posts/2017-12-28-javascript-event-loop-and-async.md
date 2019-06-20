---
layout: post
title: JavaScript事件轮询与异步机制
abstract: 单线程的JavaScript如何解决并发问题
category: technical
permalink: javascript-event-loop-and-async
author: 木逸辰
tags: [tech, javascript, event loop, callback, task queue, async]
---

### {{ page.title }}

做前端，尤其是使用过`jQuery`的同学，应该对`$.ajax`这个方法都很熟悉。这是`jQuery`封装的方法，用来请求网络资源。`ajax`即 **Asynchronous JavaScript and XML**，重点在于 **Asynchronous** 这个词，**异步**，即能够单独地另外执行任务，而不会阻塞后续代码的执行。定时器方法`setTimeout`和`setInterval`，以及`NodeJS`中的所有异步方法，都具有同样的特性，其实现原理也是一样的。

    console.log("Hello");
    setTimeout(function log(){
        console.log("and world from 1 second later");
    }, 1000);
    console.log("world");

    //Hello
    //world
    //(1 second...)
    //and world from 1 second later

现代CPU运行速度非常快，相比较起来，所有涉及到I/O的操作都太慢太慢了，因此绝大多数耗时耗资源的任务，其运行速度的瓶颈在于I/O，如文件操作、网络传输等。那么当一个涉及到I/O操作的任务在执行时，CPU实际上是闲置的，但是又被当前任务独占了，无法处理其他任务，这就造成了极大的资源浪费。对此的解决方案是多线程。当一个任务因I/O而长时间等待时，可以开启新的线程执行其他任务，CPU也不会闲置浪费。Java、C#等语言中都有线程池的概念，可以对线程(Thread)进行操作，用多线程并行执行任务来提高效率。

但JavaScript没有多线程的功能，它是单线程的，即任何时候只有一个线程，同一时间只能处理一个任务。这是因为JavaScript是作为浏览器脚本语言而设计的，它的主要用途在于为页面提供动态交互和操作DOM元素的能力。如果使用多线程的话，就会出现多个线程同时操作同一个DOM元素的情况，需要处理各种复杂的并发同步问题。为了简单高效，JavaScript被设计为单线程语言，所有代码串行执行，便不会出现并发同步的问题了。

既然JavaScript是单线程的，无法处理并发，那么对于I/O操作的阻塞怎么办呢？解决方法也简单，还是用多线程，宿主环境的多线程。代码的运行是需要环境的，Java需要JVM，C#需要CLR，JS代码也需要浏览器的解释器来执行。而宿主环境（浏览器、NodeJS）可以调用操作系统的资源，当然也可以实现多线程了。因此，解决方案就是让宿主环境来操作多个线程，使用独立的线程处理各种I/O操作，而不占用主线程的执行栈和CPU时间。这就是异步的机制了。

![event-loop](/assets/images/2017-12-28-event-loop.png)

上面这张图（来自[《Help, I'm stuck in an event-loop》](https://vimeo.com/96425312)），是浏览器的处理JavaScript的机制。可以看到，这里有两个逻辑块，一个是运行JS代码的主线程，负责执行JS代码和内存的分配管理。另一个是各种WebAPIs，处理各种异步任务。

JS开始执行后，主线程按代码顺序调用各函数，把函数压入栈中，执行，结束后再把函数推出去。当遇到需要异步处理的任务是，浏览器会把其中的异步任务分离出来，交给相应的WebAPI线程（如`setTimeout`交给timer模块，`ajax`交给网络模块，页面元素事件交给DOM模块）去做。此时主线程里相应的函数已经完成了自己的使命，被推出栈，主线程继续执行后面的代码。

而进入WebAPI的任务，由相应的独立线程执行，并保存其对应的回调函数。等到执行完毕（或者定时器时间到了），就把任务的回调函数放进任务队列(Task Queue)，等待执行。

当主线程的执行栈(Execute Stack)中的函数都执行完了，主线程就会去自动检测任务队列，看是否有等待执行的回调函数，若有，就按顺序放进执行栈执行。

        执行栈中的函数 → 完毕后检测任务队列
             ↑                ↓    
        将任务队列中最靠前的一个任务压入栈中                            

这个一直不间断的循环，就是JavaScript的事件轮询机制。通过将耗时的任务分给独立的线程模块，浏览器就实现了非阻塞(non-block)的效果。另外通过给页面元素添加监听事件，通过消息机制与回调函数结合，便使得用户能够与页面进行交互，大大提高了用户体验。

另外对于上述的轮询机制，有一个值得注意的点，就是定时器的回调函数执行时间。还是上面的代码，试想如果将定时的时间间隔设为0，也会是什么效果呢？

    console.log("Hello");
    setTimeout(function log(){
        console.log("and world from 0 second later");
    }, 0);
    console.log("world");

    //Hello
    //world
    //(0 second...)
    //and world from 0 second later

还是一样的效果，只是最后一句输出与第二句输出之间的就没有时间间隔了。通过上面的图其实很容易理解。主线程总是先执行栈中的函数，全部执行完后才会检测任务队列，而回调函数必定会先被放入任务队列，等待主线程来检测。所以上述代码会先输出"Hello"，然后将`setTimeout`交给timer模块处理，timer模块发现定时时间为0，于是立即将回调函数`log`放入任务队列，此时主线程结束`setTimeout`，开始执行下一句，输出"world"。再之后，栈中的代码执行完毕，开始检测任务队列，发现`log`函数，于是压入栈中执行，调用`console.log`方法，输出"and world from 0 second later"。

NodeJS的内部机制类似，原理都是一样的，通过宿主环境开启其他线程调用独立模块处理异步任务。不过NodeJS根据自身情况做了一些优化处理：

![node-event-loop](/assets/images/2017-12-28-node-event-loop.png)

根据上图，V8引擎解析JS代码，然后调用Node的API，而由[libuv](https://github.com/libuv/libuv)来负责为不同的任务分配不同的线程，并在任务执行完毕之后通知主线程(V8引擎)。

另外NodeJS还提供了[`process.nextTick`](https://nodejs.org/dist/latest-v8.x/docs/api/process.html#process_process_nexttick_callback_args)和[`setImmediate`](https://nodejs.org/dist/latest-v8.x/docs/api/timers.html#timers_setimmediate_callback_args)两个方法，作为对`setTimeout`和`setInterval`的补充，提供了更多回调函数的执行点，可以详细参考以加深理解。
