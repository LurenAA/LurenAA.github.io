---
title: libuv源码分析（4）async
date: 2019-11-25 02:19:34
tags: ["libuv"]
description: 从初始化到执行，讲讲libuv中async的过程。
---
#### uv_async_init
&emsp;&emsp;libuv中async的开端在uv_loop_init函数中：
```c
  //前面省略
  err = uv_async_init(loop, &loop->wq_async, uv__work_done);
  if (err)
    goto fail_async_init;

  uv__handle_unref(&loop->wq_async);
  loop->wq_async.flags |= UV_HANDLE_INTERNAL;
  //后面省略
```
&emsp;&emsp;loop->wq_async是个uv_async_t类型，它用于线程work函数调用最后处理loop->wq中的回调，暂时不用管,我在[我的第五篇文章](https://lurenaa.github.io/2019/11/26/libuv5/)会讲到它的用途。<br/>
&emsp;&emsp;我们来看uv_async_init内部：
```c
  int err;
  err = uv__async_start(loop);
  if (err)
    return err;

  uv__handle_init(loop, (uv_handle_t*)handle, UV_ASYNC);
  handle->async_cb = async_cb;
  handle->pending = 0;

  QUEUE_INSERT_TAIL(&loop->async_handles, &handle->queue);
  uv__handle_start(handle);

  return 0;
```
&emsp;&emsp;第五行以后的操作就是初始化基类uv_handle_t以及子类uv_async_t，然后将这个handle放入loop->queue(放uv_handle_t的队列)以及放入loop->async_handles（放uv_async_t的队列）中，然后uv__handle_start中将loop->active_handles加一。<br/>
&emsp;&emsp;总而言之，第五行以后的内容就是初始化uv_async_t，可以理解成**uv_async_t的构造函数**。<br/>
&emsp;&emsp;uv__async_start则不一样，它是初始化函数，它**只会调用一次**（一般情况是在uv_loop_init中调用），我们先看下它的实现：
```c
static int uv__async_start(uv_loop_t* loop) {
  int pipefd[2];
  int err;

  if (loop->async_io_watcher.fd != -1)
    return 0;

  err = uv__async_eventfd();
  if (err >= 0) {
    pipefd[0] = err;
    pipefd[1] = -1;
  }
  //中间省略

  uv__io_init(&loop->async_io_watcher, uv__async_io, pipefd[0]);
  uv__io_start(loop, &loop->async_io_watcher, POLLIN);
  loop->async_wfd = pipefd[1];

  return 0;
}
```
&emsp;&emsp;看第三行loop->async_io_watcher.fd，当你调用过一次这个函数后，loop->async_io_watcher.fd不会等于-1，以后你初始化uv_async_t类型变量，调用uv_async_init函数时，uv__async_start都是直接返回的。<br>
&emsp;&emsp;我省略掉了中间如果eventfd没有在当前系统下实现时的兼容性处理。总的来说，就是**初始化loop->async_io_watcher**。uv__io_t是为epoll设计的结构体。~~这里你肯定感觉很懵逼，请坚持一下，最后我会梳理一下总体的整个过程。~~<br>
&emsp;&emsp;uv__io_t的实现是这样的：
```c
uv__io_t{
  uv__io_cb cb;  //回调函数 
  void* watcher_queue[2]; //放入loop->watcher_queue
  void* pending_queue[2]; //同理
  unsigned int pevents; /* Pending event mask i.e. mask at next tick. */
  unsigned int events;  /* Current event mask. */
  int fd;  //文件描述符，用于epoll注册
}
```
&emsp;&emsp;这里uv__io_init函数是初始化loop->async_io_watcher这个结构体：
```c
  QUEUE_INIT(&w->pending_queue);
  QUEUE_INIT(&w->watcher_queue);
  w->cb = cb;
  w->fd = fd; //前面我们的eventfd
  w->events = 0;
  w->pevents = 0;
```
&emsp;&emsp;uv__io_start将loop->async_io_watcher放入loop->watcher_queue。还有对于loop->nfds大小的处理。
```c
  if (QUEUE_EMPTY(&w->watcher_queue))
    QUEUE_INSERT_TAIL(&loop->watcher_queue, &w->watcher_queue);

  if (loop->watchers[w->fd] == NULL) {
    loop->watchers[w->fd] = w;
    loop->nfds++;
  }
```
&emsp;&emsp;第四行以后的操作是为了在epoll后，我们得到struct event结构体，我们从event->data.fd可以得到fd，那**我们如何获取到对应的uv__io_t呢？** 就是通过loop->watchers这个数组。

#### uv_async_send
```c
int uv_async_send(uv_async_t* handle) {
  /* Do a cheap read first. */
  if (ACCESS_ONCE(int, handle->pending) != 0)
    return 0;

  /* Tell the other thread we're busy with the handle. */
  if (cmpxchgi(&handle->pending, 0, 1) != 0)
    return 0;

  /* Wake up the other thread's event loop. */
  uv__async_send(handle->loop);

  /* Tell the other thread we're done. */
  if (cmpxchgi(&handle->pending, 1, 2) != 1)
    abort();

  return 0;
}
```
&emsp;&emsp;ACCESS_ONCE：
```c
#define ACCESS_ONCE(type, var)  \
  (*(volatile type*) &(var))
```
&emsp;&emsp;这里调用一次ACCESS_ONCE，是为了告诉编译器，handle->pending可能被其他线程修改，所以别给我乱优化。<br>
&emsp;&emsp;cmpxchgi是原子操作compare_and_change。pending的有三个取值0，1，2。0代表闲置、1代表忙（比如uv_async_send调用途中）、2代表完成。loop->async_io_watcher调用uv__async_io时，会遍历loop->async_handles，通过pending来判断哪些回调该被执行。<br>
&emsp;&emsp;uv__async_send就是向loop->async_io_watcher.fd（eventfd）写（这里关系到eventfd的机制，不懂可以man eventfd）。

#### 整体调用过程
&emsp;&emsp;这里总体归纳一下async的过程。<br>
&emsp;&emsp;1.在loop_uv_init中初始化async_io_watcher，它的fd为eventfd，值为0，不可读。<br>
&emsp;&emsp;2.用户uv_async_init注册uv_async_t变量，被添加到loop->async_handles，设置回调函数。<br>
&emsp;&emsp;3.如果对uv_async_t变量调用uv_async_send，那么uv_async_t变量的pending变为2（done），并且向eventfd写，loop->async_io_watcher可读了。<br>
&emsp;&emsp;4.在uv_run的uv__io_poll中，每次都会把loop->watchers注册到epoll中，**第四步这个过程在每次事件循环中都在执行**。如果async_io_watcher的fd不可读，就没它事儿。如果可读，async_io_watcher的回调函数uv__async_io执行，它遍历loop->async_handles，将其中pending为2的uv_async_t变量移除队列，并执行其回调函数。

#### 看源码后写的小DEMO： https://github.com/LurenAA/simple_imitation_of_libuv