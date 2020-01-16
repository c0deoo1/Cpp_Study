https://stackoverflow.com/questions/9162712/what-is-the-purpose-of-epolls-edge-triggered-option

From epoll's man page:

epoll is a variant of poll(2) that can be used either as an edge-triggered
or a level-triggered interface
When would one use the edge triggered option? The man page gives an example that uses it, but I don't see why it is necessary in the example.



When an FD becomes read or write ready, you might not necessarily want to read (or write) all the data immediately.

Level-triggered epoll will keep nagging you as long as the FD remains ready, whereas edge-triggered won't bother you again until the next time you get an EAGAIN (so it's more complicated to code around, but can be more efficient depending on what you need to do).

Say you're writing from a resource to an FD. If you register your interest for that FD becoming write ready as level-triggered, you'll get constant notification that the FD is still ready for writing. If the resource isn't yet available, that's a waste of a wake-up, because you can't write any more anyway.

If you were to add it as edge-triggered instead, you'd get notification that the FD was write ready once, then when the other resource becomes ready you write as much as you can. Then if write(2) returns EAGAIN, you stop writing and wait for the next notification.

The same applies for reading, because you might not want to pull all the data into user-space before you're ready to do whatever you want to do with it (thus having to buffer it, etc etc). With edge-triggered epoll you get told when it's ready to read, and then can remember that and do the actual reading "as and when".


https://www.zhihu.com/question/20502870

epoll的边沿触发模式(ET)真的比水平触发模式(LT)快吗？(当然LT模式也使用非阻塞IO，重点是要求ET模式下的代码不能造成饥饿)

在eventloop类型(包括各类fiber/coroutine)的程序中, 处理逻辑和epoll_wait都在一个线程, ET相比LT没有太大的差别. 反而由于LT醒的更频繁, 可能时效性更好些. 在老式的多线程RPC实现中, 消息的读取分割和epoll_wait在同一个线程中运行, 类似上面的原因, ET和LT的区别不大.但在更高并发的RPC实现中, 为了对大消息的反序列化也可以并行, 消息的读取和分割可能运行和epoll_wait不同的线程中, 这时ET是必须的, 否则在读完数据前, epoll_wait会不停地无谓醒来.


LT是没读完就总是触发，如果处理的线程得不到及时的调度（比如工作线程都被打满了），epoll_wait所在的线程就会陷入疯狂的旋转。而ET是有消息来时才触发，和及时处理与否无关，频率低很多。至于使用oneshot与否是个很具体的工程问题，常规的理解是要用，但实际的选择是不用。原因在于epoll是加锁红黑树，如果每次读消息都要EPOLL_CTL_MOD，开销太大。解决方法是不设oneshot而用一个原子变量做去重，只有在原子加1前看到的是0才会启动读取线程，读取线程完成后置0。


作者：戈君
LT是没读完就总是触发，如果处理的线程得不到及时的调度（比如工作线程都被打满了），epoll_wait所在的线程就会陷入疯狂的旋转。而ET是有消息来时才触发，和及时处理与否无关，频率低很多。至于使用oneshot与否是个很具体的工程问题，常规的理解是要用，但实际的选择是不用。原因在于epoll是加锁红黑树，如果每次读消息都要EPOLL_CTL_MOD，开销太大。解决方法是不设oneshot而用一个原子变量做去重，只有在原子加1前看到的是0才会启动读取线程，读取线程完成后置0。链接：https://www.zhihu.com/question/20502870/answer/142303523
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。





ET本身并不会造成饥饿，由于事件只通知一次，开发者一不小心就容易遗漏了待处理的数据，像是饥饿，实质是bug使用ET模式.
特定场景下会比LT更快，因为它可以便捷的处理EPOLLOUT事件，省去打开与关闭EPOLLOUT的epoll_ctl（EPOLL_CTL_MOD）调用。从而有可能让你的性能得到一定的提升。
例如你需要写出1M的数据，写出到socket 256k时，返回了EAGAIN，ET模式下，当再次epoll返回EPOLLOUT事件时，继续写出待写出的数据，当没有数据需要写出时，不处理直接略过即可。
而LT模式则需要先打开EPOLLOUT，当没有数据需要写出时，再关闭EPOLLOUT（否则会一直返回EPOLLOUT事件）总体来说，ET处理EPOLLOUT方便高效些，LT不容易遗漏事件、不易产生bug如果server的响应通常较小，不会触发EPOLLOUT，那么适合使用LT，例如redis等。而nginx作为高性能的通用服务器，网络流量可以跑满达到1G，这种情况下很容易触发EPOLLOUT，则使用ET。

作者：dong
链接：https://www.zhihu.com/question/20502870/answer/89738959
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



https://www.zhihu.com/question/49741301/answer/142304853

linux的epoll_wait以及epoll_ctl是否线程安全?
场景:
线程A是一个循环, 调用epoll_wait, 当有事件发生时执行对应的回调函数.
线程B不时会建立新的连接, 使用non-block的socket, connect后调用epoll_ctl将socket加入监听.

线程A和线程B操作的是同一个epoll instance, 那么是否有潜在的问题了?
根据man page对于epoll_wait的描述:
While one thread is blocked in a call to epoll_pwait(), it is
       possible for another thread to add a file descriptor to the waited-
       upon epoll instance.  If the new file descriptor becomes ready, it
       will cause the epoll_wait() call to unblock.
按照我的理解, 前面的做法不会有问题.
但是实际程序运行过程出现了这样的现象: A线程正好从某次epoll_wait调用退出的时候, B线程加入的那个socket上发生的事件消失了(对应epoll_ctl返回值是0, 没有显示错误).
Google后得到的信息都是认为前述写法不存在问题, 但是偶然在一个github的项目的issue看到不一样的说法: epoll_wait() is oblivious to concurrent updates via epoll_ctl() · Issue #331 · cloudius-systems/osv · GitHub, 所以将原来的写法改了, B线程不能直接调用epoll_ctl, 而是写一个pipe唤醒A线程, 在A线程执行对应的操作. 改了之后bug没再出现了.
所以, 是man page的说法有误还是我理解有误??


epoll_wait和epoll_ctl都是线程安全的, 前者带acquire语意, 后者带release语意, 换句话说, 如果epoll_wait后能拿到某个新fd的事件, 那么对应的epoll_ctl前发生的内存修改都可见. 但在老版本内核中可能有bug, 比如[32/49] epoll: prevent missed events on EPOLL_CTL_MOD

作者：戈君
链接：https://www.zhihu.com/question/49741301/answer/142304853
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。


用 pipe 或 eventfd 是常规的做法，我见过的网络库都这么做。把一个pipe或是eventfd用epoll_ctl加入监听， 这样就可以随时将epoll_wait唤醒