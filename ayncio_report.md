---
typora-copy-images-to: upload
---

# Python异步I/O的总结

[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

对于游戏加速器来说，游戏线路的稳定性是十分重要的，所以导师给了我一个定时对游戏线路进行拨测的任务。我们有大概100多条静态线路，一次拨测每条线路要通过udp连接100次，记录每次从发出到接收的延迟时间，如果时间超过一定阈值则视为丢包，最后记录着一百次的平均延迟时间和丢包率。
如果使用同步的方法，这样运行一次拨测要花上大概半小时，时间过长了，所以我们考虑用异步IO来完成这个拨测。这个过程中我们尝试了两种异步方法。

  - 多线程

  - 多协程

    

### 多线程

最开始我尝试用多线程的方法来实现异步，选择操作相对简单的线程池。将一次连接线路测试封装成一个task，将所有10000多个tasks扔进线程池，设定最大线程数，很简单就完成了。

```python
from concurrent.futures import ThreadPoolExecutor

ctimer = 100
droppk_dict = {}
obuffer_dict = {}
def start_udp_ping(ping,num):
    key = ping.inpoint+','+ping.echoip
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    
    begin = datetime.datetime.now()
    end = begin
     
    data = ping.get_data()

    s.sendto(data, (ping.inpoint,int(ping.inport)) )
    s.settimeout(2)
    try :
        data, server = s.recvfrom(4096)
        end = datetime.datetime.now() 
    except Exception as e:
        print('timeout error! key:%s:'%(key))
        tmp_output.append('timeout error! key:%s'%(key))
        droppk_dict[key][num] += 1
        k = end - begin
        obuffer_dict[key][num]={'inpoint':ping.inpoint, 'echoip':ping.echoip, 'send':num, 'delay':k.total_seconds()*1000}
    s.close()

    k = end - begin
    obuffer_dict[key][num]={'inpoint':ping.inpoint, 'echoip':ping.echoip, 'send':num, 'delay':k.total_seconds()*1000}

def start_udp_ping_helper(n):
    start_udp_ping(n[0],n[1])
    
def start_all_udp_pings(all_line_list):
    global droppk_dict
    global obuffer_dict
  
    tasks = []
    for n in all_line_list:
        for num in range(1, ctimer+1):
            ping = Ping(n[0], n[1])

            tasks.append((ping,num))
    with ThreadPoolExecutor(max_worker=10) as executor:
        try:
            executor.map(start_udp_ping_helper, tasks)
        except Exception as e:
            print("executor failed: ",e)
```

但是测试结果却不尽人意，时间的确是缩短到了大约两分钟，可是很多线路出现了严重超时的情况，从而导致丢包率的误报。我尝试控制单一变量改变了一些参数，发现以下两种办法都可以有效降低延迟：

1. 减少最大线程数。 

   ```python
   with ThreadPoolExecutor(max_worker=5) as executor:
   ```

2. 在封装的task里面增加sleep函数，休息一小段时间。

   ```python
   def start_udp_ping(ping,num):
       key = ping.inpoint+','+ping.echoip
       s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
       
       begin = datetime.datetime.now()
       end = begin
        
       data = ping.get_data()
   
       s.sendto(data, (ping.inpoint,int(ping.inport)) )
       s.settimeout(2)
       time.sleep(0.5)
   ```

由此可以猜测出线路异常超时的原因是过多的连接请求同时堆砌在了IO层，单测的时候只有一个请求在等待，很快就会得到回应；可当大量请求同时发出的时候，如果一个请求要花较长时间连接，其他线路只能等待而就会造成堵塞，从而导致丢包的假象。

第一种方法是减少了一次性到达IO层的线路，从而减少堵塞率；第二种方法则是防止让每条线路一直占用资源，一定时间后就会sleep，给其他线路连接机会。可是这两种方法都有很强的弊端：1. 减少线程数同时及也会增长运行时间，在准确率和速率两者间找到较好的平衡相对较难。2. 同理，sleep也会增长运行时间，具体应该sleep的时间也难以界定。
此外，多线程会调用较多资源，对于涉及到网络连接的这种**I/O密集型**操作，多线程和多进程的资源消耗其实是没有必要的，切换线程的时候会消耗不必要的额外的CPU资源。于是在导师的指导下，我开始尝试用协程（coroutine））的方式来实现异步并发。



### 多协程

使用协程之前，我们得先了解一下什么是协程。

- [ ] A coroutine is a **function** that can suspend its execution (yield) until the given **YieldInstruction** finishes.

这是协程的官方定义，意思是可以暂时停滞（挂起)直到对应的指示结束。协程可以视作轻量级线程，多个协程只占用**一个**线程，没有切换的开销。协程也不需要多线程的锁机制，所以没有同时访问数据（race condition）的问题，执行效率比多线程会高很多。协程通过切换CPU来实现每个协程中函数的暂停与重启。不同于线程和进程的CPU切换依赖于操作系统，协程的切换是可以由我们自己控制的。所以当遇到耗时的I/O操作的时候，我们可以暂停当前的协程，让CPU去做其他的任务，等I/O操作在后台完成之后再收回控制权继续运行。因为CPU在不同的协程间切换的速度是非常快的，我们可以将其视为并发。

因为云函数只支持一些基本的库，我们最开始是希望用python原生的asyncio库来实现这个切换以达到并发的目的。

python中asyncio基本的使用方法就是先获得一个当前的**事件循环(event loop)**，然后将我们希望执行的函数定义为可等待的类型，加入循环。完成所有函数后关闭循环。而在asyncio库中，await关键字可以完成上述这个切换动作。await表示在这个地方等待子函数执行完成，把程序控制权让给其他协程。此时这个任务会主动让出event loop，去后台执行一些网络IO操作，event loop选择自己等待队列的任务继续执行。等原来网络IO的任务结束，该任务会重新加入到event loop的等待队列，等待event loop被让出从而重新调度。这样，我们就能实现在I/O端的并发了。这里要注意，await后面跟着的函数必须是**可等待（awaitable）**的。

<img src="https://github.com/qluag/asyncio_conclusion/blob/master/awaitable.png?raw=True" style="zoom:80%;" />

很自然的，我们将每条线路每一次的连接测试定义成一个协程，在def前加上async关键词就可以定义一个协程了。可是接下来注意到socket的内部函数：connect、sendto、recvfrom都不是可等待的类型，也找不到对应的可等待函数（比如tcp的read和write）。所以就算把接套字流程定义为一个协程，加入event loop，实质也是同步的，因为当碰到这些网络处理堵塞的时候协程并不会主动交出调度权。



在python的官方说明中，我们找到了一个udp的底层API：Protocol，它可以异步地执行IO端的读写工作：

<img src="https://github.com/qluag/asyncio_conclusion/blob/master/protocol.png?raw=true" alt="protocol" style="zoom:65%;" />

官方也附上了udp客户端的例子：

![](https://github.com/qluag/asyncio_conclusion/blob/master/udpclient.png?raw=true)
参考这个范例，我们也得到了能计算delay时间和出现错误增加丢包次数的自定义协议。

```python
class UdpEchoProtocol:
    def __init__(self, message, ping,key,num):
        self.message = message
        self.begin = None
        self.end = None
        self.inpoint = ping.inpoint
        self.echoip = ping.echoip
        self.key = key
        self.num = num
        self.transport = None

    def connection_made(self, transport):
        global obuffer_dict
        obuffer_dict[self.key][self.num]={'inpoint':self.inpoint, 'echoip':self.echoip, 'send':self.num, 'delay':0.0}

        self.begin = datetime.datetime.now()
        self.transport = transport
        #print('Send:', self.message)
        self.transport.sendto(self.message)

    def datagram_received(self, data, addr):
        global droppk_dict
        global obuffer_dict
        global output_temp
        self.end = datetime.datetime.now()    
        k = self.end - self.begin
        
        obuffer_dict[self.key][self.num]={'inpoint':self.inpoint, 'echoip':self.echoip, 'send':self.num, 'delay':k.total_seconds()*1000}
        output_temp.append("key=%s,send=%d,delay=%.1fms"%(self.key,self.num,round(k.total_seconds() * 1000, 1)))
        self.transport.close()

    def error_received(self, exc):
        global droppk_dict

        output_temp.append("Error received key=%s,send=%d"%(self.key,self.num))
        droppk_dict[self.key][self.num] += 1    #should be exactly 1

    def connection_lost(self, exc):
        output_temp.append("Connection closed key=%s,send=%d"%(self.key,self.num))
        pass
```

运行后总体运行时间确实大大缩短了，但是许多线路的delay时间相比正常的单测要增长了许多。我们怀疑还是之前的问题，一次性放出去的线路太多（10000+），就算是异步处理也难免会有阻塞，特别是一些本来单测时间就相对较长的线路进行等待和读写的时候。那么很自然我们改成分批次并发，一次只将一条线路的100次测试加入tasks，放入event loop运行，等这一批测试完再将新的100次测试加入新的tasks运行。这里我们会用到asyncio.gather()这个函数，它会将我们加入tasks的协程类封装成task类按顺序加入event loop运行，直到所有任务完成后返回。

<img src="https://github.com/qluag/asyncio_conclusion/blob/master/gather.png?raw=true" style="zoom:150%;" />

这样更改后虽然还是会无可避免地增加一些等待时间（大概0-20ms），但也在可接受的范围内。

延迟过长的问题解决了，接下来遇到一个整个过程中最令人头大的问题：如何控制timeout。发现这个问题的契机是出现了一条ping不通的线路。在同步单测中，recvfrom函数是可以设置timeout时间的，设置2s之后若还收不到回传就中断等待，在Exception中增加一次丢包。所以这条不通的线路正常丢包率应该是100%，每次测试时间约为2s。可是刚刚改进的程序中没有函数可以设置timeout，也就是说这个程序会一直循环地运行下去。那可以在event loop层设置循环中每个任务最长的运行时间吗？是可以的。

<img src="https://github.com/qluag/asyncio_conclusion/blob/master/wait.png?raw=True" alt="timeout" style="zoom:95%;" />

按照官方说明，wait函数可以设置timeout时间，超出时限的任务会被归到pending，没有超时的归到complete，作为task类返回。但是pending的任务不会被取消，需要手动取消。本来以为这个设置能解决我们的问题，可是实际上这个timeout并不会约束到udp protocol。原因是这样的：

```python
async def start_udp_ping(ping,loop,key,num):
    data = ping.get_data()
    return await loop.create_datagram_endpoint(lambda: UdpEchoProtocol(data,ping,key,num),remote_addr=(ping.inpoint,int(ping.inport)))
```

这个是开始一次测ping的协程函数。注意到udp协议的连接是需要先建立一个端口的，而我们加入event loop的任务其实是建立端口这个函数，没有办法直接对接到udp协议，所以超时针对的是建立端口这个动作而不是真正的接套字！（实际运行程序会发现没有一个任务会超时）所以现在任务就陷入了一个瓶颈：我们希望在socket发出包2s后若还收不到回包就打断这个动作，并进行丢包的相关处理，可现在看上去我们没有任何方法真正打断它。



那我们回到前一个节点：为什么会选择udp protocol来完成接套字呢？因为asyncio库没有高级的udp API，我们只能自定义一个协议来实现并发。那如果有一个延伸库可以像同步中使用udp协议一样在异步中完成send，receive等动作，并能强行打断不就好了？最开始因为在云函数部署新库比较麻烦所以尽量避开了这条路，现在看来是跑不掉了......不过实际操作添加新库也没有想象中那么复杂，自己撰写好一个index.py文件，将相关库的包和requirement一起打包成zip上传就可以了。经过一番搜索我们锁定了这个库：asyncio_dgram

<img src="https://github.com/qluag/asyncio_conclusion/blob/master/dgram.png?raw=True" alt="dgram" style="zoom: 50%;" />

这是它提供的函数及其使用方式：

<img src="https://github.com/qluag/asyncio_conclusion/blob/master/func.png?raw=True" alt="func" style="zoom:75%;" />

官方也提供了一个范例：

<img src="https://github.com/qluag/asyncio_conclusion/blob/master/ex.png?raw=True" alt="ex" style="zoom:67%;" />

这个是官方的说明，它提供的函数及其语法和同步操作几乎没有两样，这样我们不需要通过建立一个endpoint来连接调用函数和真正的udp连接函数了。因此，我们可以通过调用asyncio.wait()函数中的timeout来控制超时的线路了。更棒的是被打断的线路的timeout error我们可以直接在udp连接的Exception中拿到，从而直接在每个udp连接的协程中进行超时后丢包的处理。这些超时的线路会被打断然后归并到pending中，一批任务运行完我们只需要把pending的所有任务取消，然后进入下一批任务就大功告成了。

这样尝试之后又出现了一个问题，由于异步不可能做到和同步耗时完全一样，肯定会有一定程度的超时，再加上存在一些本来连接时间就比较长的线路，每次运行总会有个别线路那一批次出现多次超时，从而那一条线路的丢包率超过阈值(30%)而造成误报。既然不能避免那就把这个误差平均摊到每一条线路好了。之前我们的逻辑是每次把一条线路的100次测试放到一个tasks里面去并发运行，完成再开始下一批。那我们现在改成每一次把每一条线路的一条加入tasks，一共跑一百次，这样就算某几次并发堵塞得厉害出现较高延迟的误差，这个误差会均摊到每一条线路而不是集中在某一条，这样误差大大减小了，最后的代码如下：

```python
import random
import asyncio
import asyncio_dgram
from find_all_ping import *
from warning import *
from ping import *

# global variables setting
ctimer = 100

droppk_dict = {}
obuffer_dict = {}
tmp_output = []

# 运行一条线路的一次测试
async def start_udp_ping(ping,num):  
    key = ping.inpoint+','+ping.echoip

    # asyncio_dgram
    s = await asyncio_dgram.connect((ping.inpoint,int(ping.inport)))
    
    begin = datetime.datetime.now()
    end = begin
     
    data = ping.get_data()

    await s.send(data)
    try :
        data, addr = await s.recv()
        end = datetime.datetime.now() 
    except Exception as e:
        print('timeout error! key:%s:'%(key))
        tmp_output.append('timeout error! key:%s'%(key))
        droppk_dict[key][num] += 1
        k = end - begin
        obuffer_dict[key][num]={'inpoint':ping.inpoint, 'echoip':ping.echoip, 'send':num, 'delay':k.total_seconds()*1000}
    s.close()

    k = end - begin
    obuffer_dict[key][num]={'inpoint':ping.inpoint, 'echoip':ping.echoip, 'send':num, 'delay':k.total_seconds()*1000}

    return

# 运行所有的线路
async def start_all_udp_pings(all_line_list):
    global droppk_dict
    global obuffer_dict
    
    for num in range(1, ctimer+1):
        tasks = []
        # 一批运行所有的线路
        for n in all_line_list:
            ping = Ping(n[0], n[1])
            key = ping.inpoint + "," + ping.echoip

            droppk_dict[key][num] = 0
            obuffer_dict[key][num] = {}
            tasks.append(asyncio.ensure_future(start_udp_ping(ping,num)))
        # settimeout: 3s
        completed, pending = await asyncio.wait(tasks, timeout = 3)
        print('num:%s, completed:%d, pending:%d'%(num, len(completed), len(pending)))
        tmp_output.append('num:%s, completed:%d, pending:%d'%(num, len(completed), len(pending)))
        if pending:
            for t in pending:
                print('canceling tasks')
                tmp_output.append('canceling tasks')
                t.cancel()

def run_test(): 
    global droppk_dict
    global total_send_dict
    global obuffer_dict
    global tmp_output

    all_line_list = find_all_ping()
    # remove thge abnoral one
    all_line_list.remove(['118.89.127.211', '150.109.196.7'])
    #all_line_list.remove(['106.55.74.179', '119.28.157.166'])
    #all_line_list = shuffle(all_line_list)
    #all_line_list = all_line_list[:1]
    random.shuffle(all_line_list)

    for n in all_line_list:
        ping = Ping(n[0], n[1])
        key = ping.inpoint + "," + ping.echoip

        droppk_dict[key] = {}
        obuffer_dict[key] = {}

    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    try:
        loop.run_until_complete(start_all_udp_pings(all_line_list))
    finally:
        loop.close()

    f = open("/tmp/result.txt", "w")
    for text in tmp_output:
        f.write(text+'\n')
    f.close()   

    for n in all_line_list:
        ping = Ping(n[0], n[1])
        key = ping.inpoint+','+ping.echoip
        #loop.close()
        #print(obuffer_dict[key])

        # adjust the order of output
        # warn if exceed the threshold
        droppk_count = 0
        delay_bar = 0
        rate_final = 0
        # sort droppk ascending
        droppk_sorted = {k: v for k, v in sorted(droppk_dict[key].items(), key=lambda item: item[1])}
        droppk_dict[key] = droppk_sorted

        send = 1
        output = ["ASYNC: inpoint:%s echoip:%s"%(ping.inpoint,ping.echoip)]    
        for k in droppk_dict[key].keys():
            #print(k)
            droppk_count += droppk_dict[key][k]

            output.append("send=%d,droppk=%d,delay=%f ms,rate="%(send,droppk_count,obuffer_dict[key][k]['delay'])+"{:.1%}".format(droppk_count/float(send)))
            send += 1
            delay_bar += obuffer_dict[key][k]['delay']
        rate_final = droppk_count/float(ctimer)
        if droppk_count != ctimer:
            delay_bar /= (ctimer-droppk_count)

        output.append("\n")
        output_handler(output,delay_bar,rate_final,ping.echoip)
        warning_handler(ping.inpoint,ping.echoip,delay_bar,rate_final)

    # upload to COS
    cos_upload("/tmp/result.txt")

    return
```

最后，异步确实是个比较复杂的部分，debug也相对较难，经常有点虚无缥缈的感觉......以后还要继续学习和总结才行~