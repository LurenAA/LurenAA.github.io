---
title: libuv源码分析（5）uv_fs_*
date: 2019-11-26 02:57:15
tags: ["libuv"]
description: 讲讲uv_fs_*的调用过程
---
#### uv_fs_*
&emsp;&emsp;uv_fs_*这一系列的函数基本是一致的，它们的逻辑大概是如下：
```c
//x代表一种操作open、write等
int uv_fs_x(...uv_fs_t* req...) {
  INIT(x); //uv_fs_t和其基类uv_req_t的基本初始化
  ... //这里是每个操作各自不同对于req的初始化
  POST; //提交这个任务
}
```

#### INIT
&emsp;&emsp;INIT这个宏定义函数没有特别的地方，就是把req初始化，该置0的置0。

#### POST
&emsp;&emsp;其实现如下：
```c
#define POST                          
  do {   //dowhile包裹作用域          
    if (cb != NULL) {                 
      uv__req_register(loop, req);    
      uv__work_submit(loop,           
      &req->work_req,                 
      UV__WORK_FAST_IO,               
      uv__fs_work,                    
      uv__fs_done);                   
      return 0;                       
    }                                 
    else {                            
      uv__fs_work(&req->work_req);    
      return req->result;             
    }                                 
  }                                   
  while (0)
```
&emsp;&emsp;这里通过有无回调函数来决定调用同步版本还是异步版本。
> http://docs.libuv.org/en/v1.x/fs.html
libuv provides a wide variety of cross-platform sync and async file system operations. All functions defined in this document take a callback, which is allowed to be NULL. If the callback is NULL the request is completed synchronously, otherwise it will be performed asynchronously.

&emsp;&emsp;uv__fs_work这个函数就是**文件操作的封装**，所有的文件操作都通过这个函数来完成，即使是异步，最终也要在别的线程中同步执行这个函数。 <br>
&emsp;&emsp;uv__fs_done这个函数会调用用户给的回调函数，这个函数会在uv_run中的is_poll函数中得到执行。

&emsp;&emsp;uv__work_submit函数的实现是这样的：
```c
void uv__work_submit(uv_loop_t* loop,struct uv__work* w,
enum uv__work_kind kind, void (*work)(struct uv__work* w),
void (*done)(struct uv__work* w, int status)) 
{
  uv_once(&once, init_once);
  w->loop = loop;
  w->work = work;
  w->done = done;
  post(&w->wq, kind);
}
```
&emsp;&emsp;uv_once(&once, init_once);是初始化多个线程，我在[我的第三篇文章](https://lurenaa.github.io/2019/11/18/libuv3/)中有介绍。不过当时对于子线程运行的worker函数没有提及，work函数大概是这样的：
```c
static void worker(void* arg) {
  ...
  uv_mutex_lock(&mutex);
  for (;;) {
    while (QUEUE_EMPTY(&wq)...) {
      idle_threads += 1;
      uv_cond_wait(&cond, &mutex);
      idle_threads -= 1;
    }

    q = QUEUE_HEAD(&wq);
    ...

    QUEUE_REMOVE(q);
    QUEUE_INIT(q);  
    ...

    w = QUEUE_DATA(q, struct uv__work, wq);
    w->work(w);

    uv_mutex_lock(&w->loop->wq_mutex);
    w->work = NULL;  
    QUEUE_INSERT_TAIL(&w->loop->wq, &w->wq);
    uv_async_send(&w->loop->wq_async);
    uv_mutex_unlock(&w->loop->wq_mutex);

    uv_mutex_lock(&mutex);
    ...
  }
}
```
&emsp;&emsp;我去掉了对于slow_io的处理，大致是这样一个过程。

&emsp;&emsp;一开始线程会卡在uv_cond_wait这里，直到被uv_cond_signal唤醒，如果唤醒时wq队列中有任务，它就会执行任务，w->work(w)也就是调用uv__fs_work。然后把w放入loop->wq（为了uv__fs_done的执行）。

&emsp;&emsp;uv_async_send调用让loop->wq_async可读，主线程就从uv_run中的uv__io_poll的epoll_pwait中醒来，wq_async的回调函数会遍历loop->wq执行w->done。（[我的第四篇文章](https://lurenaa.github.io/2019/11/25/libuv4/)有讲这一部分的详细内容）

#### 谁来触发uv_cond_signal唤醒子线程呢？
🥣uv__work_submit中的post函数：
```c
uv_mutex_lock(&mutex);
...
QUEUE_INSERT_TAIL(&wq, q);
if (idle_threads > 0)
  uv_cond_signal(&cond);
uv_mutex_unlock(&mutex);
```
&emsp;&emsp;我再次省略了slow_io的部分，因为它们只是特殊处理。

&emsp;&emsp;该函数有空闲的线程就唤醒，不然就阻塞该线程。