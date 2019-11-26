---
title: libuv源码分析（1）事件循环分析
date: 2019-11-16 16:45:44
tags: ["libuv"]
description: 初步阅读libuv，了解事件循环的过程。
---

#### 前言

> &emsp;&emsp;libuv总是报出一些让人难以理解的错误😂，作为一个C的项目，不具有Java、JavaScript、php那样的人气，很难百度到一些问题的答案，甚至google也不行。为了用好libuv，也为了学习吧。我开始看libuv的源码，不知道自己能走多远。。。

#### 事件循环

<img src="https://img-blog.csdnimg.cn/20191116165040610.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMDE1MDQ4,size_16,color_FFFFFF,t_70" width="400" height="500" alt="Event Loop" align=center>

这是官方事件循环的示意图。[链接->官方图片位置](http://docs.libuv.org/en/v1.x/design.html)
```cpp
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int ran_pending;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  while (r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop);
    ran_pending = uv__run_pending(loop);
    uv__run_idle(loop);
    uv__run_prepare(loop);

    timeout = 0;
    if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);

    uv__io_poll(loop, timeout);
    uv__run_check(loop);
    uv__run_closing_handles(loop);

    if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
  }

  /* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
}

```
&emsp;&emsp;整个事件循环就是在主线程的uv_run（）调用中执行的。我就跟着官方的介绍一步一步来看（[官方介绍](http://docs.libuv.org/en/v1.x/design.html)）。

##### 第一步
> The loop concept of ‘now’ is updated. The event loop caches the current time at the start of the event loop tick in order to reduce the number of time-related system calls.

&emsp;&emsp;第一步是更新时间。对应代码如下：

```cpp
  uv__update_time(loop);
```
&emsp;&emsp;总结来说就是调用这个函数，更新时间。~~uv__update_time实现我下一篇来介绍~~ 

##### 第二步

> If the loop is alive an iteration is started, otherwise the loop will exit immediately. So, when is a loop considered to be alive? If a loop has active and ref’d handles, active requests or closing handles it’s considered to be alive.

```cpp
r = uv__loop_alive(loop);
```
&emsp;&emsp;用uv__loop_alive函数获取loop状态。
&emsp;&emsp;如果uv__loop_alive返回零或者loop->stop_flag == 1说明loop终止，直接跳过循环，到代码最下面（~~这里有一些性能的处理暂时不管~~ ），退出：

```cpp
/* The if statement lets gcc compile it to a conditional store. Avoids
   * dirtying a cache line.
   */
  if (loop->stop_flag != 0)
    loop->stop_flag = 0;

  return r;
```
&emsp;&emsp;loop->stop_flag == 0的一个来源是调用了uv_stop，这个函数在手册中看见。它的源代码也很清晰。
```cpp
void uv_stop(uv_loop_t* loop) {
  loop->stop_flag = 1;
}
```
&emsp;&emsp;如果loop状态OK，那么就进入循环中。

##### 第三步

> Due timers are run. All active timers scheduled for a time before the loop’s concept of now get their callbacks called.

&emsp;&emsp;对应代码这一部分：

```cpp
uv__run_timers(loop);
其实现：
void uv__run_timers(uv_loop_t* loop) {
  struct heap_node* heap_node;
  uv_timer_t* handle;

  for (;;) {
    heap_node = heap_min(timer_heap(loop));
    if (heap_node == NULL)
      break;

    handle = container_of(heap_node, uv_timer_t, heap_node);
    if (handle->timeout > loop->time)
      break;

    uv_timer_stop(handle);
    uv_timer_again(handle);
    handle->timer_cb(handle);
  }
}
```
&emsp;&emsp;将堆里面已经超时的拿出来运行。
##### 第四步

> Pending callbacks are called. All I/O callbacks are called right after polling for I/O, for the most part. There are cases, however, in which calling such a callback is deferred for the next loop iteration. If the previous iteration deferred any I/O callback it will be run at this point.

对应：

```cpp
ran_pending = uv__run_pending(loop);
其实现：
static int uv__run_pending(uv_loop_t* loop) {
  QUEUE* q;
  QUEUE pq;
  uv__io_t* w;

  if (QUEUE_EMPTY(&loop->pending_queue))
    return 0;

  QUEUE_MOVE(&loop->pending_queue, &pq);

  while (!QUEUE_EMPTY(&pq)) {
    q = QUEUE_HEAD(&pq);
    QUEUE_REMOVE(q);
    QUEUE_INIT(q);
    w = QUEUE_DATA(q, uv__io_t, pending_queue);
    w->cb(loop, w, POLLOUT);
  }

  return 1;
}

```
&emsp;&emsp;将loop->pending_queue中的任务拿出来运行。
##### 第五、六、九步

>5.Idle handle callbacks are called. Despite the unfortunate name, idle handles are run on every loop iteration, if they are active

>  6.Prepare handle callbacks are called. Prepare handles get their callbacks called right before the loop will block for I/O.

> 9.Check handle callbacks are called. Check handles get their callbacks called right after the loop has blocked for I/O. Check handles are essentially the counterpart of prepare handles.

```cpp
  uv__run_idle(loop);
  uv__run_prepare(loop);
  uv__run_check(loop);
```
&emsp;&emsp;这三部为什么要一起说呢？因为它们的实质是一样的。在每次循环固定的位置调用。
&emsp;&emsp;这三个函数定义在loop-watcher.c这个文件里面，它们是用宏定义定义的。只改了idle、prepare、check这三个名字的部分，其余部分函数都是一样的。

```cpp
/* Copyright Joyent, Inc. and other Node contributors. All rights reserved.
 *
 * Permission is hereby granted, free of charge, to any person obtaining a copy
 * of this software and associated documentation files (the "Software"), to
 * deal in the Software without restriction, including without limitation the
 * rights to use, copy, modify, merge, publish, distribute, sublicense, and/or
 * sell copies of the Software, and to permit persons to whom the Software is
 * furnished to do so, subject to the following conditions:
 *
 * The above copyright notice and this permission notice shall be included in
 * all copies or substantial portions of the Software.
 *
 * THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 * IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 * FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 * AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 * LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
 * FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS
 * IN THE SOFTWARE.
 */

#include "uv.h"
#include "internal.h"

#define UV_LOOP_WATCHER_DEFINE(name, type)                                    \
  int uv_##name##_init(uv_loop_t* loop, uv_##name##_t* handle) {              \
    uv__handle_init(loop, (uv_handle_t*)handle, UV_##type);                   \
    handle->name##_cb = NULL;                                                 \
    return 0;                                                                 \
  }                                                                           \
                                                                              \
  int uv_##name##_start(uv_##name##_t* handle, uv_##name##_cb cb) {           \
    if (uv__is_active(handle)) return 0;                                      \
    if (cb == NULL) return UV_EINVAL;                                         \
    QUEUE_INSERT_HEAD(&handle->loop->name##_handles, &handle->queue);         \
    handle->name##_cb = cb;                                                   \
    uv__handle_start(handle);                                                 \
    return 0;                                                                 \
  }                                                                           \
                                                                              \
  int uv_##name##_stop(uv_##name##_t* handle) {                               \
    if (!uv__is_active(handle)) return 0;                                     \
    QUEUE_REMOVE(&handle->queue);                                             \
    uv__handle_stop(handle);                                                  \
    return 0;                                                                 \
  }                                                                           \
                                                                              \
  void uv__run_##name(uv_loop_t* loop) {                                      \
    uv_##name##_t* h;                                                         \
    QUEUE queue;                                                              \
    QUEUE* q;                                                                 \
    QUEUE_MOVE(&loop->name##_handles, &queue);                                \
    while (!QUEUE_EMPTY(&queue)) {                                            \
      q = QUEUE_HEAD(&queue);                                                 \
      h = QUEUE_DATA(q, uv_##name##_t, queue);                                \
      QUEUE_REMOVE(q);                                                        \
      QUEUE_INSERT_TAIL(&loop->name##_handles, q);                            \
      h->name##_cb(h);                                                        \
    }                                                                         \
  }                                                                           \
                                                                              \
  void uv__##name##_close(uv_##name##_t* handle) {                            \
    uv_##name##_stop(handle);                                                 \
  }

UV_LOOP_WATCHER_DEFINE(prepare, PREPARE)
UV_LOOP_WATCHER_DEFINE(check, CHECK)
UV_LOOP_WATCHER_DEFINE(idle, IDLE)

```


##### 第七步

> Poll timeout is calculated. Before blocking for I/O the loop calculates for how long it should block. These are the rules when calculating the timeout:
> If the loop was run with the UV_RUN_NOWAIT flag, the timeout is 0.
If the loop is going to be stopped (uv_stop() was called), the timeout is 0.
If there are no active handles or requests, the timeout is 0.
If there are any idle handles active, the timeout is 0.
If there are any handles pending to be closed, the timeout is 0.
If none of the above cases matches, the timeout of the closest timer is taken, or if there are no active timers, infinity.

```cpp
if ((mode == UV_RUN_ONCE && !ran_pending) || mode == UV_RUN_DEFAULT)
      timeout = uv_backend_timeout(loop);
```
&emsp;&emsp;这部分是取决于uv_run的模式的特殊处理，暂时不细看。

##### 第八步

> The loop blocks for I/O. At this point the loop will block for I/O for the duration calculated in the previous step. All I/O related handles that were monitoring a given file descriptor for a read or write operation get their callbacks called at this point.

```cpp
uv__io_poll(loop, timeout);
```
&emsp;&emsp;这一部分对于不同操作系统有所不同，linux是poll，mac是kqueue。

##### 第十步

> Close callbacks are called. If a handle was closed by calling uv_close() it will get the close callback called.

```cpp
 uv__run_closing_handles(loop);
```
&emsp;&emsp;调用各类的close回调函数。

##### 第十一、十二步

> 11.Special case in case the loop was run with UV_RUN_ONCE, as it implies forward progress. It’s possible that no I/O callbacks were fired after blocking for I/O, but some time has passed so there might be timers which are due, those timers get their callbacks called.
> 12.Iteration ends. If the loop was run with UV_RUN_NOWAIT or UV_RUN_ONCE modes the iteration ends and uv_run() will return. If the loop was run with UV_RUN_DEFAULT it will continue from the start if it’s still alive, otherwise it will also end.

&emsp;&emsp;对于uv_run不同模式的一点特殊处理。

```cpp
if (mode == UV_RUN_ONCE) {
      /* UV_RUN_ONCE implies forward progress: at least one callback must have
       * been invoked when it returns. uv__io_poll() can return without doing
       * I/O (meaning: no callbacks) when its timeout expires - which means we
       * have pending timers that satisfy the forward progress constraint.
       *
       * UV_RUN_NOWAIT makes no guarantees about progress so it's omitted from
       * the check.
       */
      uv__update_time(loop);
      uv__run_timers(loop);
    }

    r = uv__loop_alive(loop);
    if (mode == UV_RUN_ONCE || mode == UV_RUN_NOWAIT)
      break;
```


#### 小结
&emsp;&emsp;宏观上梳理一下整个事件循环的过程。