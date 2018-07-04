---
layout: post
title: "简述python异步i/o库 —— asyncio"
date: 2017-10-31 16:12:02 +0800
categories: python
tags: asyncio
---

python的asyncio库以协程为基础，event_loop作为协程的驱动和调度模型。该模型是一个单线程的异步模型，类似于node.js。下图我所理解的该模型

<!-- more -->

![](https://user-gold-cdn.xitu.io/2017/10/31/23b17e8d92ec76fcefb7abadc0101dd7)

事件循环通过select()来监听是否存在就绪的事件，如果存在就把事件对应的callback添加到一个task list中。然后从task list头部中取出一个task执行。在单线程中不断的注册事件，执行事件，从而实现了我们的event_loop模型。

**event_loop中执行的task并不是函数**

如果**我们把上图当成一个web服务器**，左边的一个task当成一次http请求需要执行的完整任务。如果我们每一次run_task()都执行完一个完整的任务，再去run下一个task。 那这跟普通的串行服务器并没有区别。在并发环境下造成的用户体验非常差。

> 具体怎么差你可以脑补一下，毕竟我们现在是使用单线程方式实现的web服务器

所以task如果对应一个完整的http请求那么其不可能是一个函数，因为函数需要从头执行到尾占用着整个线程。那你觉得task是什么呢？

如果你不知道答案的话可以看一看我的另一篇文章 [简述python的yield和yield from](http://www.weiwenhao.xyz/article/the-yield-and-yield-of-python-the-from)

没错，task是一个generator，或者可以叫做可中断的函数。task的代码依旧是从上写到下来处理一个http请求。也就是我们所说的同步的代码组织。

但是有所不同的是，在task中，我们遇到i/o操作时，我们就把i/o操作交给selector（稍后我们解析一下selector，并且把该i/o操作准备完毕后需要执行的回调也告诉selector。然后我们使用yield保存并中断该函数。

此时线程的控制权回到event_loop手中。event_loop首先看一下selector中是否存在就绪的数据，存在的话就把对应的回调放到task list的尾部（如图），然后从头部继续run_task()。

你可能想问上面中断的task什么时候才能继续执行呢？我前一句说过了，event_loop每一次循环都会检测selector中是否存在就绪的i/o操作，如果存在就绪的i/o操作，我们对应就把callback放到task的尾部，当event_loop执行到这个task时。我们就能回到我们刚刚中断的函数继续执行啦，而且此时我们需要的i/o操作得到的数据也已经准备好了。

> 这种操作如果你站在函数的角度会有种神奇的感觉，在函数眼里，自己需要get遥远服务器的一些数据，于是调动get()，然后瞬间就得到了遥远服务器的数据。没错在函数的眼里就是瞬间得到，这感觉就仿佛是穿越到了未来一样。

你可能又想问，为什么把callback放到task，然后run一下就回到原有的函数执行位置了？

这我也不知道，我并没有深追asyncio的代码，这对于我来说有些复杂。但如果是我的话，我只要在callback中设置一个变量gen指向我们的generator就行了，然后只要在callback中`gen.send(res_data)`，我们就能回到中断处继续执行了。如果你有兴趣的话可以自己使用debug来追一下代码。

不过我更推荐你阅读一下这篇博文 [深入理解 Python 异步编程(上)](http://python.jobbole.com/88291/)

### 这里还有几个问题。

比如我们在task中需要执行一个1+2+3+到2000万这样一个操作，这个操作耗时有些长，而且不属于i/o操作，没法交给selector去调度，此时我们需要自己yield，让其他的task能有机会来使用我们唯一的线程。这样就又有一个新的问题。yield后，我们什么时候再次来执行这个被中断的函数呢？

问题代码示例

```python
import asyncio

def print_sum():
    sum = 0
    count = 0
    for a in range(20000000):
        sum += a
        count += 1
        if count > 1000000:
            count = 0
            yield
    print('1+到2000万的和是{}'.format(sum))

@asyncio.coroutine
def init():
    yield from print_sum()

loop = asyncio.get_event_loop()
loop.run_until_complete(init())
loop.run_forever()
```



我想我们可以这样，把这个中断的task直接加入到task list的尾部，然后继续event_loop，这样让其他task有机会执行，并且处理起来更加的简单。 asyncio库也确实是这样做的。

*但是asyncio还提供了更好的做法，我们可以再启动一个线程来执行这种cpu密集型运算*

---



再来看看另外一个问题。如果在一个凌晨三点半，你task list此时是空的，那么你的event_loop怎么运作？继续不停的loop等待新的http请求进来？ no，我们不允许如此浪费cpu的资源。asyncio库也不允许。

首先看两行event_loop中的代码片段，也就是上图中右上角部分的select(timeout)部分

```python
	event_list = self._selector.select(timeout)
    self._process_events(event_list)
```

> 补充一点，作为一台web服务器，我们总是需要socket()、bind()、listen()、来创建一个监听描述符sockfd，用来监听到来的http请求，与http请求完成三路握手。然后通过accept()操作来得到一个已连接描述符connectfd。
>
> 这里的两个文件描述符，此时都存在于我们的系统中，其中sockfd继续用来执行监听http请求操作。已经连接了的客户端我们则通过connectfd来与其通信。一般都是一个sockfd对多个connectfd。
>
> 更多的细节推荐阅读  ——《unix网络编程卷一》中的关于socket编程的几章

asyncio对于网络i/o使用了 selector模块，selector模块的底层则是由 epoll()来实现。也就是一个同步的i/o复用系统调用**（你定会惊讶于asyncio的竟然使用了同步i/o来实现？我们在下一节来解读一下epoll函数）**

*这里你可以去读一下python手册中的selector模块，看看这个模块的作用*

epoll()函数有个timeout参数，用来控制该函数是否阻塞，阻塞多久。映射到高层就是我们上面的`selector.select(timeout)`中的timeout。原来我们的event_loop中的存在一个timeout。这样凌晨三点半我们如何处理event_loop我想你已经心里有数了吧。

asyncio的实现和你想的差不多。如果task list is not  None那么我们的timeout=0也就是非阻塞的。解释一下就是，我们调用selector.select(timeout = 0 )，该函数会马上返回结果，我们对结果做一个上面讲过的处理，也就是`self._process_events(event_list)`。然后我们继续run task。

如果我们的task list is None， 那么我们则把timeout=None。也就是设置成阻塞操作。此时我们的代码或者说线程会阻塞在selector.select(timeout = 0)处，换句话说就是等待该函数的返回。**当然这样做的前提是，你往selector中注册了需要等待的socket描述符。**

---

还有一些其他的问题，比如异步mysql是如何在asyncio的基础上实现的，这可能需要去阅读aiomysql库了。

你也许发现，我们一旦使用了event_loop实现单线程异步服务器，我们写的所有代码就都不是我们来控制执行了，代码的执行权全部交给了event_loop，event_loop在适当的时间run task。读过廖雪峰python教程的小伙伴一定看过这句话

> 这就是异步编程的一个原则：一旦决定使用异步，则系统每一层都必须是异步，“开弓没有回头箭”。  

这就是异步编程。

---

你也许对asyncio的作用，或者使用，或者代码实现有着很多的疑问，我也是如此。但是很抱歉，我并不怎么熟悉python，也没有使用asyncio做过项目，只是出于好奇所以我对python的异步i/o进行了一个了解。

我是一个纸上谈兵的门外汉，到最后我也没能看清asyncio库的具体实现。我接下来的计划中并不打算对asyncio库进行更多的研究，但是我又不甘心这两天对asyncio库的研究付诸东流。所以我留下这篇博文，算是对自己的一个交待！希望下次能够有机会，能够更加了解python和asyncio的前提下，再写一篇深入解析python—asyncio的博文。
