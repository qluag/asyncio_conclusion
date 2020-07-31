---
typora-copy-images-to: upload
---

# Python实现I/O端并发编程的总结

[TOC]

## 背景

对于游戏加速器来说，游戏线路的稳定性是十分重要的，所以导师给了我一个定时对游戏线路进行拨测的任务。

我们有大概100多条静态线路，每条线路包括一对本地IP和其对应的虚拟IP (inport, echoport)，一次拨测就是对这些线路进行100次通过UDP协议，从客户端发出请求和接收回包的测试，记录每次从发出发出请求到成功接收的延迟时间，延迟超过一定阈值或收不到回包则视为丢包；最后记录每条线路这一百次的平均延时和丢包率，超过相应阈值的线路则被归为异常线路，将被推送到微信机器人进行告警。

这是单测一条线路的代码（后文的单测代码都指代此代码，细节处理可能有出入）:

```python
import datetime
import socket

def calc_ping(num, inpoint,inport,echoip,echoport,droppk,delay_list):
    
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM) # 创建UDP接套字

    s.sendto(data, (inpoint,int(inport)) ) # data是之前包装好的数据，发送到指定地址
    s.settimeout(2) # 设置超时时间，超过2s视为丢包
    try :
        data, server = s.recvfrom(4096) # 尝试接收回包
        end = datetime.datetime.now() 
    except Exception as e:
        droppk=droppk+1 # 丢包次数加一
        
    k = end - begin # 计算一次测试的延迟时间
    delay_list.append(round(k.total_seconds() * 1000, 1))

    if num == ctimer:
        delay_sum = 0.0
        for i in range(len(delay_list)):
            delay_sum += delay_list[i]
        delay_avg = round(delay_sum / len(delay_list), 1) #计算平均延迟时间

    return droppk          
            
def main_handler(event, context):
    # 主函数
    droppk = 0 # 全局变量，总的丢包次数
    ctimer = 100 # 测试100次
    delay_list = []
    delay_avg = 0
    for num in range(1,int(ctimer)+1):
        droppk = calc_ping(num, inpoint, inport, echoip, echoport, droppk, delay_list)
    droprate=droppk/float(ctimer) # 计算丢包率
    print("平均延迟： "+"{:.1f}".format(delay_avg)+"s, 丢包率 "+ "{:.1%}".format(droprate))
```

完整的拨测就是对所有100多条线路进行这样一次单测，最开始的时候我们用的是无脑串行的方法：用for循环完成所有线路的测试。果不其然，这样一趟运行下来要花大约30分钟，而我们希望每5分钟就能进行一次拨测，这样实在是太慢了......很自然的，我们考虑并发编程来提高代码速率。



## Python并发编程

首先简单解释一下什么是并发。并发的实质是一个物理CPU(也可以多个物理CPU) 在若干道程序之间多路复用，通过操作系统的各种任务调度算法，实现用多个任务“一起”执行[1]。并发实际上并不是真的所有任务都同时在执行，但是由于CPU切换任务的速度相当快，在我们看来就像是在一起执行了。

为什么串行100条线路时间要花那么久呢？因为这里的I/O操作是**阻塞（blocking）**的，在进行I/O操作的时候CPU空闲着傻傻等待，直到获取回应才会继续接下来的工作。比如一次socket的读操作流程大概如下：

<img src="https://nesoy.github.io/assets/posts/20170127/Blocking.jpg" style="zoom:70%;" />

打个比方，I/O层就像是平底锅，I/O操作是煎蛋，人是CPU。我们想吃煎蛋，我们也想看电视，串行下我们一直等着蛋煎好了才会去看电视，可实际上我们完全可以在蛋没煎熟的时候先去看电视，等煎好了再去拿蛋，这就是并发。并发有很多种方式，出于多线程并发操作简洁的特点，我们首先尝试了这种方法。



### 多线程并发（Multithreading）

我们尝试用线程池的方法来实现多线程并发。每进行一条线路的拨测，我们将100次测试包装成一个tasks，放入线程池执行指定函数，设定最大线程数，很简单就完成了。为什么是每一条线路进行一次并发而不是将所有线路的全部测试都包装成tasks一次性并发呢？因为每次单测我们建立socket进行读写操作，最后销毁socket都是需要网络开销的。如果一次性发送过多请求，在网络上传输的数据就会过多而导致传输速度变慢，最后造成发出请求和收到回包之间的延迟时间过长而误报（我们最开始尝试一次性并发100多条线路发现延时几乎是正常一次单测的10倍）。

线程池实现多线程并发大概的代码如下：

```python
from concurrent.futures import ThreadPoolExecutor

ctimer = 100 # 每条线路测试100次
def start_udp_ping(ping,num):
    # 进行上文代码中的一次单测
    
def start_all_udp_pings(all_line_list):
    # all_line_list是所有本地ip和虚拟ip对（inport，echoport）的集合
    for n in all_line_list:
        tasks = []
        for num in range(1, ctimer+1):
            ping = Ping(n[0], n[1]) # Ping是我们自定义的一个类，包含固定端口和发包的相关信息，                                         
                                    # 初始化后可以通过内部函数生成线路对应的发包数据
        with ThreadPoolExecutor(max_workers=10) as executor: # 允许最多同时运行10条线程
            try:
                executor.map(start_udp_ping, tasks) # 将tasks中的参数传入单测函数，放入线程池
            except Exception as e:
                print("executor failed: ",e)
```

但是测试结果却不尽人意，时间的确是缩短到了大约两分钟，可是很多线路出现了延时超过阈值的情况，从而导致丢包率的误报。并发一定是会比正常运行一次单测花的等待时间要长一点的，可长这么多甚至超过阈值确实不应该的。更奇怪的是一些时间较短的线路基本没有受到影响，每次测试都是固定的几条线路中的一两条超时。我们控制单一变量改变了一些参数，发现以下两种办法都可以有效降低延迟：

1. 减少最大线程数。 

   ```python
   with ThreadPoolExecutor(max_worker=5) as executor:
   ```

2. 在单测函数里面增加sleep函数，休息一小段时间。

   ```python
   def start_udp_ping(ping,num):
       # 单测函数，在最后获得回包后挂起0.5s
       time.sleep(0.5)
   ```

这着实有点诡异，我到现在也只能猜测是因为这几条线路比较特殊，可能是回包数据较大或者什么别的原因，kernel在处理他们的读写操作时网络开销对于网络请求数更敏感，可能同时存在几十个请求就会使其传输速率大大变慢。而减少允许同时运行的最大线程数和暂时挂起完成任务的线程，都可以一定程度上减少同时存在的网络请求数目，所以延迟时间的问题得以缓和。但这究竟是不是背后真正的原因，笔者也不是很清楚，欢迎各位读者思考讨论！

可是这两种解决方法都有很强的弊端：1. 减少线程数同时及也会增长运行时间，在准确率和速率两者间找到较好的平衡相对较难。2. 同理，sleep也会增长运行时间，具体应该sleep的时间也难以界定。此外，多线程并发在切换线程的时候会消耗不必要的额外的CPU资源，对于涉及到网络的这种**I/O密集型**操作，多线程的资源消耗其实是没有必要的。所以我们试图用不用额外CPU开销的更优方案完成拨测。



### I/O多路复用（Multiplexing）

I/O多路复用通过使用select，poll，epoll等对socket进行监控。当客户端调用了select，那么整个进程会被阻塞，与此同时kernel会“监视”所有select负责的socket，当任何一个socket中的数据准备好了，select就会返回。这个时候用户进程再调用读取操作，将数据从kernel拷贝到用户进程[2]。它的原理如图：

<img src="https://images.cnblogs.com/cnblogs_com/xuexianqi/1753541/o_200428130755%E5%A4%9A%E8%B7%AF%E5%A4%8D%E7%94%A8IO.png" style="zoom:52%;" />

继续拿上文的煎鸡蛋做比喻，I/O多路复用就好比你一下在锅里打了好几个蛋，然后每个蛋熟了之后会发出”滋滋“的声音。打完蛋我们就看电视去了，一旦听到有蛋熟了的“滋滋”声就去拿蛋吃掉，然后继续回去看电视。

但是由于I/O多路复用时基于回调函数的，与我们习惯的常规思维不通，比较难理解，我们并没有采取这一方法，这里也不做过多的描述了。于是在导师的指导下，我开始尝试用**协程（coroutine）**的方式来实现并发。



### 多协程并发（Multi Coroutines）

#### （一）初识

使用协程之前，我们先了解一下什么是协程：

*A coroutine is a **function** that can suspend its execution (yield) until the given **YieldInstruction** finishes.*

这是协程的官方定义，意思是可以暂时挂起直到对应的指示结束的函数。也就是说不同于一般函数，我们可以控制协程的中断，挂起，恢复，回调。以下是一些协程相比于线程的优点：

1. 多个协程只占用一个线程，没有线程切换的开销。
2. 因为只有一个线程，协程也不需要多线程的锁机制，所以没有同时访问数据（race condition）的问题，执行效率比多线程会高很多。
3. 不同于线程和进程的CPU切换依赖于操作系统，协程的切换是可以由程序员自己控制的。

多协程并发的模型图如下：

<img src="https://bbsmax.ikafan.com/static/L3Byb3h5L2h0dHBzL2ltZzIwMTguY25ibG9ncy5jb20vYmxvZy8xNDk0NDM4LzIwMTkwMy8xNDk0NDM4LTIwMTkwMzAxMDkxNDU4NDAyLTc1MDMwOTA3OS5wbmc=.jpg" style="zoom:100%;" />

继续煎蛋的例子。这里就相当于你打完蛋后雇了个保姆帮你看着，然后就跑去看电视了，蛋熟了之后保姆会叫你回去拿蛋。

我们最终是将这个拨测程序部署到云函数上的，因为云函数只支持一些基本的Python库，我们最开始是希望用Python的原生库来实现多协程并发的。早在Python2.2的时候就有了基于生成器（generator）实现协程的方法，生成器函数最大的特点是可以接受外部传入的一个变量，并根据变量内容计算结果后返回。Python2.5中生成器引入了send和yield。yield类似于Java中的return，用于返回一个值，但是不同的是下次运行函数的时候是从yield后面的代码开始运行的，因为yield关键字标记了一个函数为生成器函数，所以用一个next方法进行迭代，碰到yield后就不继续执行了，并返回这个yield后面的值[3]。例子：

```python
def generator():
    print("hello")
    yield "I pend myself, them return" # 这里函数会暂时将自己挂起，获得返回值后继续运行
    print("keeping executing!")
```

在 Python3.3 中，生成器又引入了yield from，实现在生成器内调用另外生成器的功能。Python3.5开始引入了async和await关键字（位于ayncio库），从语义上定义了原生协程关键字，避免了使用者对生成器协程与生成器的混淆。

由于云函数使用的是**Python3.6**版本，所以我们将使用asyncio库实现多协程并发。

python中asyncio基本的使用方法就是先获得一个当前的**事件循环(event loop)**，然后将我们希望执行的函数定义为**可等待（awaitable）**的类型，加入循环。完成所有函数后关闭循环。

那么什么是事件循环，什么又是可等待的类型？他们又该如何使用呢？

下图是协程，事件循环和策略之间的相互作用：

<img src="https://media.springernature.com/original/springer-static/image/chp%3A10.1007%2F978-1-4842-4401-2_2/MediaObjects/470771_1_En_2_Fig1_HTML.jpg" style="zoom:42%;" />



##### 事件循环

事件循环是 asyncio 应用的核心。 事件循环会运行异步任务和回调，执行网络 IO 操作，以及运行子进程。他的作用实际上就是对我们希望执行的任务进行管理调度，在主程序中循环执行，不断捕捉当前应该执行的任务，当任务被挂起时将执行权按顺序让出给空闲的任务[4]。下图阐释了事件循环监听的原理：

<img src="https://ithelp.ithome.com.tw/upload/images/20181007/20107274shJ5sIoujK.png" style="zoom:80%;" />

让我们用一个例子来感受一下事件循环的使用方法：

```python
import asyncio
loop = asyncio.get_event_loop() # 获得当前的event loop

async def fun1(): # 定义一个耗时的协程
    print("Start fun1 coroutine.")
    await asyncio.sleep(3) # 中断协程3s
    print("Resume fun1 coroutine.")

async def fun2(): 
    print("Start fun2 coroutine.")
    # do something
    print("Finish fun2 coroutine.")

tasks = [ # 建立一个任务列表
    asyncio.ensure_future(fun1()),
    asyncio.ensure_future(fun2()),
]

loop.run_until_complete(asyncio.wait(tasks))
# 将fun1和fun2两个协程注册到事件循环中
# 启动事件循环，按顺序先运行fun1，遇到await关键词挂起，切换到fun2，等fun2运行结束后切回到fun1
loop.close() # 关闭事件循环
# output:
# Start fun1 coroutine.
# Start fun2 coroutine.
# Finish fun2 coroutine.
# Resume fun1 coroutine.
```

其中asyncio.sleep就是一个可等待类，我们可以发现事件循环其实就像一双大手，操纵着协程的执行、暂停、恢复。除了上述例子中用到的建立get_event_loop和运行run_until_complete事件循环的方法，还有new_event_loop、set_event_loop、run_forever等，大家可以去Python官方查看使用方法。

##### 可等待类

而在asyncio库中，await关键字用于完成我们讨论过的线程执行权切换动作。await表示在这个地方暂时将自身挂起，等待子函数执行完成，把程序控制权让给其他协程。此时这个任务会主动让出event loop，去后台执行一些网络IO之类的操作，event loop选择自己等待队列的任务继续执行；等原来网络IO的任务结束，该任务会重新加入到event loop的等待队列，等待event loop被让出从而重新调度。这样，我们就能实现在I/O端的并发了。这里要注意，await后面跟着的函数必须是**可等待（awaitable）**的。

官方定义只有三种类型是可以等待的：协程类，任务类和未来类。

- ###### **协程（Coroutine）**

  在def前加上async关键词就可以将这个函数定义为协程函数，协程可以在其他协程中被等待。举个例子：

  ```python
  import asyncio
  
  async def say_after(delay, what):
      await asyncio.sleep(delay) # 暂停delay对应的秒数
      print(what) 
  
  async def main():
      await say_after(1, 'hello')
      await say_after(2, 'world')
  
  asyncio.run(main())
  # output
  # hello
  # world
  ```

- ###### **任务（Task）**

  Task就是对Coroutine类进行了一下包装，被用来设置日程以便于并发执行协程。我们通过asyncio.create_task()（python3.7之前为asyncio.ensure_future()）等函数将协程包装为一个任务，那么该协程将会自动排入日程并准备运行。用Task类来运行上述的函数的话：

  ```python
  async def main():
      task1 = asyncio.create_task(
          say_after(1, 'hello'))
  
      task2 = asyncio.create_task(
          say_after(2, 'world'))
      
      await task1
      await task2
      # 等待直到两个任务都运行完毕
  
  asyncio.run(main())
  # output
  # hello
  # world
  ```

- ###### **未来（future）**

  Future是一种特殊的底层可等待对象，它代表异步操作中最终返回的结果。当一个 Future 对象被等待，这意味着协程将保持等待状态直到该Future对象在其他地方操作完毕，通常情况下我们没有必要在应用层级的代码中创建Future对象。举个例子：
  
  ```python
  import ayncio
  
  async def fun():
      return 22
  
  async def main():
      loop = asyncio.get_event_loop() # 获取当前事件循环
  
      fut =  loop.run_in_executor(None, fun) # 创造一个Future类对象
      await fut # 开始运行协程
      print(fut.result()) # 打印出最后返回的结果
      # 除此之外也可以对这个Future对象添加完成后的回调 (add_done_callback)、
      # 取消任务 (cancel)、设置最终结果 (set_result)、设置异常 (set_exception) 等
  
  asyncio.run(main()) 
  # output:
  # 22
  ```

##### 协程并发

要实现多个函数的并发，我们必需运用asyncio.gather()/asyncio.wait()把这些函数包装成可等待的类型加入日程。gather和wait还有一点细微的区别，gather类似一个黑盒，它注重于按给定的顺序收集并收集协程的结果；而wait更低级一点，它不会直接给出结果，它只是等待，直到获取返回的结果这个条件被满足。我们通过代码来说明一下区别：

```python
async def fun1():
    print('Suspending 1')
    await asyncio.sleep(2)
    print('Resuming 1')
    return 1

async def fun2():
    print('Suspending 2')
    await asyncio.sleep(1)
    print('Resuming 2')
    return 2
```

```python
# asyncio.gather
tasks = [asyncio.ensure_future(fun1()), asyncio.ensure_future(fun2())]
value1, value2 = loop.run_until_complete(asyncio.gather(*tasks)) # 获得返回的结果
print((value1, value2))

# output
# Suspending 1
# Suspending 2
# Resuming 2
# Resuming 1
# (1,2)
```

```python
# asyncio.wait
tasks = [fun1(), fun2()]
done, pending = loop.run_until_complete(asyncio.wait(tasks)) # 返回完成和暂停的协程
print(done)
print(pending)

# output
# Suspending 1
# Suspending 2
# Resuming 2
# Resuming 1
# {<Task finished coro=<fun1() done, defined at <ipython-input-5-5ee142734d16>:1> result=1>,
#  <Task finished coro=<fun2() done, defined at <ipython-input-5-5ee142734d16>:8> result=2>}
```



#### （二）尝试

了解这些信息之后，很自然的，我们可以将每条线路每一次的连接测试定义成一个协程，然后用asyncio.gather()/asyncio.wait()加入事件循环实现并发。可是接下来注意到与UDP接套字有关的函数：connect()、send()、recvfrom()，这些都不是可等待的类型，在asyncio也找不到对应的可等待类（比如TCP对应的read()和write()就是可等待的）。所以就算把UDP接套字流程定义为一个协程，加入event loop，实质上也是同步的，因为当碰到这些网络处理堵塞的时候协程并不会主动交出调度权，而是等待直到IO操作完成。

虽然没有找到对应的可以直接调用的高级可等待类函数，在python的官方说明中，我们找到了一个UDP的底层API：BaseProtocol，它可以异步地执行IO端的读写工作。而在Protocol这个大类下面分有四个小类：Protocol，BufferedProtocol，DatagramProtocol以及SubprocessProtocol。我们可以参考这些协议的内部结构，自定义一个适合我们程序使用的协议。

##### 基础协议（Base Protocol）

基础协议（Base Protocol）是asyncio提供的应用于网络协议的抽象类，其中所有的内部函数都是可回调的。值得注意的是，协议类必须和传输（transport）类一起使用，一个基础协议的方法应该由其所对应的传输调用。

##### 传输（Transport）

传输和协议用于底层事件循环接口的使用，在最顶层，传输只关心**怎样**传送字节内容，而协议决定传送**哪些**字节内容(还要在一定程度上考虑何时传送)。传输和协议总是存在着一一对应的关系，共同定义网络I/O和进程间I/O的抽象接口。

我们发现其中DatagramProtocol是专门应用于UDP连接的。参考官方的范例，我们也得到了能计算delay时间和出现错误增加丢包次数的自定义的协议。

运行后总体运行时间确实大大缩短了，但是许多线路的delay时间相比正常的单测要增长了许多。我们怀疑还是之前的问题，一次性放出去的线路太多（10000+），就算是异步处理也难免会有阻塞，特别是一些本来单测时间就相对较长的线路进行等待和读写的时候，那么被打断的协程就不能及时得到恢复，从而造成多余的等待时间。那么很自然我们改成分批次并发，一次只将一条线路的100次测试加入tasks，放入event loop运行，等这一批测试完再将新的100次测试加入新的tasks运行。这里我们用到asyncio.gather()这个函数，如上文所说，它会将我们加入tasks的可等待类函数按顺序加入event loop运行，直到所有任务完成后返回。

```python
# 运行所有的线路
async def start_all_udp_pings(all_line_list):
    # all_line_list：所有本地ip和虚拟ip对（inport，echoport）的集合
    for n in all_line_list:
        tasks = []
        # 一批运行所有的线路
        for num in range(1, ctimer+1): # ctimer=100，一条线路测试100次
            tasks.append(asyncio.ensure_future(start_udp_ping(ping,num)))
            # 加入一次单测的协程
        await asyncio.gather(tasks)
```

这样更改后虽然还是会无可避免地增加一些等待时间（大概0-20ms），但也在可接受的范围内。



#### （三）困境

延迟过长的问题解决了，接下来遇到一个整个过程中最令人头大的问题：如何控制timeout。

发现这个问题的契机是出现了一条ping不通的线路。在单测中，recvfrom()函数是可以设置timeout时间的，设置2s之后若还收不到回传就中断等待，在Exception中增加一次丢包。所以这条ping不通的线路正常丢包率应该是100%，每次测试时间约为2s。可是刚刚改进的程序中没有函数可以设置timeout，也就是说这个程序会一直循环地运行下去。那可以在event loop层设置循环中每个任务最长的运行时间吗？是可以的。

按照官方说明，wait函数可以设置timeout时间，超出时限的任务会被归到pending，没有超时的归到complete，作为一对task类返回。但是pending的任务不会被取消，需要手动取消。所以我们舍弃gather，用wait取而代之来实现并发:

```python
# 运行所有的线路
async def start_all_udp_pings(all_line_list):
    # all_line_list：所有本地ip和虚拟ip对（inport，echoport）的集合
    for n in all_line_list:
        tasks = []
        # 一批运行所有的线路
        for num in range(1, ctimer+1): # ctimer=100，一条线路测试100次
            tasks.append(asyncio.ensure_future(start_udp_ping(ping,num)))
            # 加入一次单测的协程
        completed, pending = await asyncio.wait(tasks, timeout = 3)
        
        if pending:
            # 超时取消
            for t in pending:
                print('canceling tasks')
                tmp_output.append('canceling tasks')
                t.cancel()
```

本来以为这个替换能解决我们的问题，可是实际上这个timeout并不会约束到协议类。原因是这样的：

```python
async def start_udp_ping(ping,loop,key,num):
    data = ping.get_data()
    return await loop.create_datagram_endpoint(lambda: UdpEchoProtocol(data,ping,key,num),remote_addr=(ping.inpoint,int(ping.inport)))
```

这个是开始一次测ping的协程函数。注意到udp协议的连接是需要先建立一个端口的，实际上这个create_datagram_endpoint()就是我们上文在传输中提到的底层事件循环接口。而我们加入event loop的任务其实是建立端口这个底层函数，没有办法直接对接到UDP协议，所以超时针对的是建立端口这个动作而不是真正的接套字（实际运行程序会发现没有一个任务会超时）！所以现在任务就陷入了一个瓶颈：我们希望在socket发出包2s后若还收不到回包就打断这个动作，并进行丢包的相关处理，可现在看上去我们没有任何方法能真正打断它。



#### （四）解决

那我们回到前一个节点：为什么会选择基础协议来完成接套字呢？因为asyncio库没有高级的UDP API，我们只能自定义一个基础协议来实现并发。那如果有一个第三方库可以像同步中使用UDP协议一样在异步中完成send，receive等动作，并能强行打断不就好了？最开始因为在云函数部署新库比较麻烦所以尽量避开了这条路，现在看来是跑不掉了......不过实际操作添加新库也没有想象中那么复杂，自己撰写好一个index.py文件，将相关库的文件包和注明版本的requirement.txt一起打包成zip上传就可以了。经过一番搜索我们锁定了这个库：**asyncio_dgram**。

这是官方提供的函数及其使用方式：

<img src="https://github.com/qluag/asyncio_conclusion/blob/master/func.png?raw=True" alt="func" style="zoom:75%;" />

根据官方的说明，它提供的函数及其语法和同步操作中接套字的函数几乎没有两样，这样我们不需要通过建立一个底层事件循环接口来连接调用函数和真正的UDP接套字了。因此，我们也可以通过调用asyncio.wait()函数中的timeout来控制超时的线路了。更棒的是被打断的线路的timeout error这个信号我们可以直接在UDP接套字的Exception中接收到，从而直接在每个udp连接的协程中进行超时后丢包的处理。而之前的BaseProtocol，超时的信号没有办法得到拦截。最后这些超时的线路会被打断然后归并到pending中，等一批任务运行完，我们只需要把pending的所有任务取消，然后进入下一批任务就大功告成了。

这样尝试之后又出现了一个问题，由于异步不可能做到和同步耗时完全一样，肯定会有一定程度的超时，再加上存在一些本来连接时间就比较长的线路，实际操作中每次运行总会有个别线路的那一批次出现多次超时，从而导致那一条线路的丢包率超过阈值(30%)而造成误报。

既然不能避免那就把这个误差平均摊到每一条线路好了。之前我们的逻辑是每次把一条线路的100次测试放到一个tasks里面去并发运行，完成再开始下一批。那我们现在改成每一次把每一条线路的一条加入tasks，一共跑一百次，这样就算某几次并发堵塞得厉害出现较高延迟的误差，这个误差会均摊到每一条线路而不是集中在某一条，这样误差大大减小了，最后这部分的代码如下：

```python
# 运行所有的线路
async def start_all_udp_pings(all_line_list): 
    # all_line_list：所有本地ip和虚拟ip对（inport，echoport）的集合
    for num in range(1, ctimer+1): #ctimer=100，一条线路测试100次
        tasks = []
        # 一批运行所有的线路
        for n in all_line_list:
            tasks.append(asyncio.ensure_future(start_udp_ping(ping,num)))
            # 加入一次单测的协程
        completed, pending = await asyncio.wait(tasks, timeout = 3)
        # 获得完成和因为超时暂停的任务
        
        if pending:
            # 取消超时的任务
            for t in pending:
                print('canceling tasks')
                tmp_output.append('canceling tasks')
                t.cancel()
```

除了asyncio，asyncio_dgram之外，Python还有一些与协程相关的库，大家感兴趣可以去深入了解一下：async_timeout, asyncore, Twisted, Tornado, Gevent 。

最后，异步确实是个比较复杂的部分，debug也相对较难，经常有点虚无缥缈的感觉......以后还要继续学习和总结才行~



## 参考文献

[1] 涤生手记（2017年12月6日）。《开发要搞清楚什么是并发，并行，串行，同步，异步？》。取自https://blog.csdn.net/qq_26442553/article/details/78729793

[2] historyasamirror（2010年7月31日）。《IO - 同步，异步，阻塞，非阻塞 （亡羊补牢篇）》。取自https://blog.csdn.net/historyasamirror/article/details/5778378?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param

[3] imkobedroid（2020年4月16日）。《Python协程、yield、yield from》。取自https://www.jianshu.com/p/6a56f8f7be0a

[4] L·Y（2020年4月27日）。《理解 Python 的 asyncio（二）：事件循环》。取自https://imliyan.com/blogs/article/%E7%90%86%E8%A7%A3%20Python%20%E7%9A%84%20asyncio%EF%BC%88%E4%BA%8C%EF%BC%89%EF%BC%9A%E4%BA%8B%E4%BB%B6%E5%BE%AA%E7%8E%AF/