---
layout: post
title: "生产者消费者问题"
categories: blog
excerpt: "生产者消费者问题是多线程同步问题的一个经典案例，该问题描述了两类共享固定大小缓冲区的线程（或进程），即生产者和消费者在实际运行过程中发生的问题。"
tags: [多线程, 操作系统]
share: true
image:
  feature:
date: 2017-07-26T23:04:37-08:00
modified: 
---

注：具体生产者消费者问题的详细概念可以参考[Kaiwii的专栏](http://blog.csdn.net/kaiwii/article/details/6758942)和[爱拼才会赢](http://blog.chinaunix.net/uid-21411227-id-1826740.html)两位作者的博文。我就是通过这两篇博文想明白这个问题的，在此也向两位作者表示感谢。

生产者和消费者是两类共享固定大小缓冲区的线程，生产者主要作用是生成一定量的数据放入缓冲区，然后重复此过程。与此同时，消费者也从缓冲区取出一定量的数据进行消费。该问题的关键就是要保证生产者不会在数据区满的时候放入数据，而消费者不会在数据区空的时候获取数据。

以下代码是我采用C++11提供的多线程库自己写出来的demo，有问题欢迎大家联系我指正^_^。
```
#include<iostream>
#include<thread>
#include<mutex>
#include<condition_variable>
#include<queue>

using namespace std;

condition_variable conp;   //生产者条件变量
condition_variable conc;   //消费者条件变量
mutex mtx;                 //关键代码段互斥锁
queue<int> msg;            //消息队列
bool flag = true;          //生产者是否还需要生产消息
#define msgQueueMaxNum 3   //消息队列大小
#define msgAllNum 30       //生产者需要生产的消息总数
int msgPos = 0;            //生产者正在生产的消息序号


void producer()
{
    while (flag)
    {
        unique_lock<mutex> lck(mtx);
        conp.wait(lck, []{return msg.size() != msgQueueMaxNum;});
        msg.push(msgPos);
        cout << "producer " << this_thread::get_id() << " produce msg:" << msg.back() << "(p)." << endl;
        ++msgPos;
        if (msgAllNum - 1 == msgPos)
            flag = false;
        conc.notify_one();
    }
}

void consumer()
{
    while (true)
    {
        unique_lock<mutex> lck(mtx);
        conc.wait(lck, []{return msg.size() != 0;});
        cout << "consumer " << this_thread::get_id() << " consume msg:" << msg.front() << "(c)." << endl;
        msg.pop();
        conp.notify_one();
    }
}

int main()
{
    thread pro1(producer);
    thread pro2(producer);
    thread con1(consumer);
    thread con2(consumer);
    pro1.join();
    pro2.join();
    con1.join();
    con2.join();
    return 0;
}
```

运行效果如下：
```
producer 2 produce msg:0(p).
consumer 5 consume msg:0(c).
producer 2 produce msg:1(p).
producer 2 produce msg:2(p).
producer 2 produce msg:3(p).
consumer 5 consume msg:1(c).
consumer 5 consume msg:2(c).
consumer 5 consume msg:3(c).
producer 2 produce msg:4(p).
producer 2 produce msg:5(p).
producer 2 produce msg:6(p).
consumer 4 consume msg:4(c).
consumer 4 consume msg:5(c).
consumer 4 consume msg:6(c).
producer 3 produce msg:7(p).
producer 3 produce msg:8(p).
producer 3 produce msg:9(p).
consumer 5 consume msg:7(c).
consumer 5 consume msg:8(c).
consumer 5 consume msg:9(c).
producer 2 produce msg:10(p).
producer 2 produce msg:11(p).
producer 2 produce msg:12(p).
consumer 5 consume msg:10(c).
consumer 5 consume msg:11(c).
consumer 5 consume msg:12(c).
producer 3 produce msg:13(p).
producer 3 produce msg:14(p).
producer 3 produce msg:15(p).
consumer 4 consume msg:13(c).
consumer 4 consume msg:14(c).
consumer 4 consume msg:15(c).
producer 2 produce msg:16(p).
producer 2 produce msg:17(p).
consumer 4 consume msg:16(c).
consumer 4 consume msg:17(c).
producer 2 produce msg:18(p).
producer 2 produce msg:19(p).
producer 2 produce msg:20(p).
consumer 5 consume msg:18(c).
consumer 5 consume msg:19(c).
consumer 5 consume msg:20(c).
producer 3 produce msg:21(p).
producer 3 produce msg:22(p).
producer 3 produce msg:23(p).
consumer 5 consume msg:21(c).
consumer 5 consume msg:22(c).
consumer 5 consume msg:23(c).
producer 3 produce msg:24(p).
producer 3 produce msg:25(p).
producer 3 produce msg:26(p).
consumer 4 consume msg:24(c).
consumer 4 consume msg:25(c).
consumer 4 consume msg:26(c).
producer 3 produce msg:27(p).
producer 3 produce msg:28(p).
consumer 5 consume msg:27(c).
consumer 5 consume msg:28(c).
producer 2 produce msg:29(p).
consumer 5 consume msg:29(c).

```