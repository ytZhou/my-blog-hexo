---
title: Redis源码分析（一）：aeMain()函数概述
date: 2016-05-29 11:18:02
categories:
- Code
tags:
- Linux 
- C/C++ 
- Lock 
- Redis 
- Key-Value 
- Source Code
---
## 缘起
临近毕业大部分事情处理的差不多，只想安静的看点东西，思考一下。找来了redis源码配合《Redis设计与实现》看了一下。大概浏览了一遍，看到main函数的最后其实就是aeMain()当中的一个大循环，对这个大循环有些兴趣。这不由得让我想起之前了一部分的Linux Kernel代码，Kernel源码最后其实也是一个大的循环，根据进程调度算法选择进程运行，模式还是比较相似。当然，Kernel源代码肯定要比这个复杂 ^_^
写下这篇博文也算是对学习过程的一个记录和思考。

## 对NoSQL的接触
在接触redis之前，最初接触到的比较成熟NoSQL是叫做Tokyo Cabinet的一套Key-Value存储（在这之前看过国人写的一个叫做Hash DB的Key-Value）。之前也看过它当中的hashdb的实现，Tokyo Cabinet与redis不同之处就是Tokyo Cabinet是一个持久化的Key-Value存储，每个db都会对应一个磁盘上持久化的文件。
继续本文正题。^_^

## redis.c中的大循环
从下面的代码可以看到redis最后其实就是一个大的循环。只要redis服务不停止就不断调用beforeSleep()和aeProcessEvents（）函数。
{% codeblock lang:c %}
//redis.c
int main()
{
    //................
    aeSetBeforeSleepProc(server.el,beforeSleep);
    aeMain(server.el);
    aeDeleteEventLoop(server.el);
    return 0;
}
{% endcodeblock %}

{% codeblock lang:c %}
//ae.c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
{% endcodeblock %}

## beforesleep函数“注册”
从下面的代码可以看到aeSetBeforeSleepProc()函数将beforeSleep“注册”到server.el下。
其实就是让server.el.beforsleep这个函数指针指向beforesleep函数。
{% codeblock lang:c %}
void aeSetBeforeSleepProc(aeEventLoop *eventLoop, aeBeforeSleepProc *beforesleep) {
     eventLoop->beforesleep = beforesleep;
}
{% endcodeblock %}

## beforeSleep(eventLoop)
beforeSleep主要有以及几个功能：

1. 如果启动集群功能（在我自己搭建过程是standalone mode，并未开启集群模式。其实是只有一台机器，玩不了集群模式→_→），则通过clusterBeforeSleep()进行集群故障转移，状态更新等操作；
2. 调用activeExpireCycle()处理“过期”的Key-Value对，减少内存的占用
3. 如果在上一次循环中有客户端blocked，那么给所有的slave发送一个ACK请求；
4. 将AOF buffer写入到磁盘当中。有趣的是，AOF buffer写入磁盘是另外的线程去完成，并未使用主进程去完成，避免主进程的阻塞和挂起，相比内存操作而言磁盘要慢一些。这一个点在后面的文章中去分析。
{% codeblock lang:c %}
void beforeSleep(struct aeEventLoop *eventLoop) {
    REDIS_NOTUSED(eventLoop);
    
    if (server.cluster_enabled) clusterBeforeSleep();
    
    if (server.active_expire_enabled && server.masterhost == NULL)
        activeExpireCycle(ACTIVE_EXPIRE_CYCLE_FAST);

    if (server.get_ack_from_slaves) {
        robj *argv[3];

        argv[0] = createStringObject("REPLCONF",8);
        argv[1] = createStringObject("GETACK",6);
        argv[2] = createStringObject("*",1); 
        replicationFeedSlaves(server.slaves, server.slaveseldb, argv, 3);
        decrRefCount(argv[0]);
        decrRefCount(argv[1]);
        decrRefCount(argv[2]);
        server.get_ack_from_slaves = 0;
    }

    if (listLength(server.clients_waiting_acks))
        processClientsWaitingReplicas();

    if (listLength(server.unblocked_clients))
        processUnblockedClients();

    flushAppendOnlyFile(0);
}
{% endcodeblock %}

## aeProcessEvents(eventLoop, AE_ALL_EVENTS)
aeProcessEvents()函数的主要功能简单来说，就是调用epoll_wait获取状态改变的fd，判断满足要求的fd的，并调用指定的处理函数。处理完成client的请求之后转而去运行server.el.timeEventHead中的事件函数。这样就完成了一次大的aeMain()中的循环。
{% codeblock lang:c %}
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            long now_sec, now_ms;

            /* Calculate the time missing for the nearest
             * timer to fire. */
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            tvp->tv_sec = shortest->when_sec - now_sec;
            if (shortest->when_ms < now_ms) {
                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
                tvp->tv_sec --;
            } else {
                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
            }
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
            } else {
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                tvp = NULL; /* wait forever */
            }
        }

        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; 
}
{% endcodeblock %}


## 与aeMain()相关的数据结构
如下图所示，redis中主要的数据结构。在后续的文章中会重点分析跟aeMain()比较相关的部分是如何完成初始化，以及aeMain(）中网络编程相关的部分。


## 本文参考
《redis设计与分析》
《Redis in Action》


