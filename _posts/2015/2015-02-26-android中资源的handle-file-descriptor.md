---
layout: post
title: "Android中资源的handle——file descriptor"
categories:
- 
tags:
- 


---
Android系统中有很多资源需要跨进程共享，比如说BufferQueue中的buffer，以及与之一一对应的Fence。共享Buffer的时候，肯定不可能将Buffer内容复制一遍到对方进程，其开销不是系统能承受的，最合理的应该是利用MMU的特性，就可以轻易地将同一块buffer映射到不同的进程地址空间。我们在从Gralloc模块分配出来buffer后最先拿到的是它的handle，通过handle能唯一地找到这块buffer，即这个handle和这块buffer一一对应。然后我们可以把这个handle跨进程传输给另外的进程，那些进程可以利用该handle将buffer映射到自己的地址空间。Android中定义的native_hanle_t：

{% highlight c++ %}
typedef struct native_handle
{
    int version;        /* sizeof(native_handle_t) */
    int numFds;         /* number of file-descriptors at &data[0] */
    int numInts;        /* number of ints at &data[numFds] */
    int data[0];        /* numFds + numInts ints */
} native_handle_t;
{% endhighlight %}

显然，拷贝一个handle比拷贝整个buffer的开销要小得多。

从结构体定义中可以看到，既可以用file descriptors（FD）来表示资源，也可以用系统中唯一的一个整数（ID）来标识资源。比如说，可以用ID来表示buffer，用FD表示Fence，合起来组成一个handle。要着重说明的是使用FD的方法。为此，Android的Binder传输框架特意设计了一种传输类型——文件描述符（其他四种是：强/弱Binder，强/弱引用）。其基本实现也很简单：FD在进入驱动层传输的时候转化为一个struct file结构体指针，接收进程中调用get_unused_fd()拿到一个空闲的fd，然后将该指针放到自己的file table中。于是，两个进程中各自的FD可以索引到同一个file结构体，通过file结构体的私有指针就可以找到具体的资源了。

用ID的方法则简单得多，因为ID是全局的，所以在一个进程中如果某块buffer的ID是5，在另一个进程自然也是5。接触过linux图形驱动的人可能知道，这种方法在GEM——Graphics Execution Manager中有个特定的名字叫做——Flink，而用FD的方法则称之为——Prime。当前linux桌面上的DRI2框架用就是Flink方法。

那这两种方法有什么区别呢？最重要的一点是安全性。如果用全局ID的话，第三方进程可以投石问路，随意猜测一个ID，来试探系统中的资源，可以比较轻易的窥视甚至改写某个资源的内容。而用FD的方法就安全得多，因为FD的传输是由比较严格的系统编程接口控制。如果没有申请共享资源，就随意猜测FD，只会让内核抱怨：在当前进程没有与该FD对应的打开文件！所以对于一些重要的资源，我们应该偏向于用FD来标识它们。