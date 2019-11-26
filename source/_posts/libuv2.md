---
title: libuv源码分析（2）uv__loop_alive
date: 2019-11-16 17:58:11
tags: ["libuv"]
description: 上一篇说了一下整体的事件循环，对于UV_RUN_DEFAULT模式来调用uv_run来说，uv__loop_alive就决定了是否退出，这一篇看一下uv__loop_alive的源码。
---

##### 前言
&emsp;&emsp;上一篇说了一下整体的事件循环，对于UV_RUN_DEFAULT模式来调用uv_run来说，uv__loop_alive就决定了是否退出，这一篇看一下uv__loop_alive的源码。

##### 详情

```cpp
static int uv__loop_alive(const uv_loop_t* loop) {
  return uv__has_active_handles(loop) ||
         uv__has_active_reqs(loop) ||
         loop->closing_handles != NULL;
}
```
&emsp;&emsp;可见loop的状态取决于三个方面：handles、reqs、closing_handles 

##### handles
&emsp;&emsp;uv__has_active_handles就是检查loop->active_handles值是否大于0.

```cpp
#define uv__has_active_handles(loop)                                          \
  ((loop)->active_handles > 0)
```


```cpp
/* Handle types. */
typedef struct uv_loop_s uv_loop_t;
typedef struct uv_handle_s uv_handle_t;
typedef struct uv_dir_s uv_dir_t;
typedef struct uv_stream_s uv_stream_t;
typedef struct uv_tcp_s uv_tcp_t;
typedef struct uv_udp_s uv_udp_t;
typedef struct uv_pipe_s uv_pipe_t;
typedef struct uv_tty_s uv_tty_t;
typedef struct uv_poll_s uv_poll_t;
typedef struct uv_timer_s uv_timer_t;
typedef struct uv_prepare_s uv_prepare_t;
typedef struct uv_check_s uv_check_t;
typedef struct uv_idle_s uv_idle_t;
typedef struct uv_async_s uv_async_t;
typedef struct uv_process_s uv_process_t;
typedef struct uv_fs_event_s uv_fs_event_t;
typedef struct uv_fs_poll_s uv_fs_poll_t;
typedef struct uv_signal_s uv_signal_t;
```
&emsp;&emsp;handles列表如上。handle在调用时，会包含一个函数的调用，就是
uv__handle_start。下图所示，是哪些函数调用了uv__handle_start。~~有一些handle不在其中，可能与其调用方式有关，我暂时无法解释~~ 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191116181231294.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMDE1MDQ4,size_16,color_FFFFFF,t_70)
```cpp
#define uv__handle_start(h)                                                   \
  do {                                                                        \
    if (((h)->flags & UV_HANDLE_ACTIVE) != 0) break;                          \
    (h)->flags |= UV_HANDLE_ACTIVE;                                           \
    if (((h)->flags & UV_HANDLE_REF) != 0) uv__active_handle_add(h);          \
  }                                                                           \
  while (0)
```
&emsp;&emsp;uv__handle_start函数在调用时，会调用uv__active_handle_add，uv__active_handle_add就是将loop->active_handles++

```cpp
#define uv__active_handle_add(h)                                              \
  do {                                                                        \
    (h)->loop->active_handles++;                                              \
  }                                                                           \
  while (0)
```
&emsp;&emsp;相应的在handle结束时有uv__active_handle_rm的调用，(h)->loop->active_handles减一。

```cpp
#define uv__active_handle_rm(h)                                               \
  do {                                                                        \
    (h)->loop->active_handles--;                                              \
  }                                                                           \
  while (0)
```

##### req
&emsp;&emsp;uv__has_active_reqs和handle的道理一样，是检测(loop)->active_reqs.count > 0。active_reqs是个共用体，它的另一个用途暂时我还不知道。

```cpp
#define uv__has_active_reqs(loop)                                             \
  ((loop)->active_reqs.count > 0)
```

```cpp
/* Request types. */
typedef struct uv_req_s uv_req_t;
typedef struct uv_getaddrinfo_s uv_getaddrinfo_t;
typedef struct uv_getnameinfo_s uv_getnameinfo_t;
typedef struct uv_shutdown_s uv_shutdown_t;
typedef struct uv_write_s uv_write_t;
typedef struct uv_connect_s uv_connect_t;
typedef struct uv_udp_send_s uv_udp_send_t;
typedef struct uv_fs_s uv_fs_t;
typedef struct uv_work_s uv_work_t;
```
&emsp;&emsp;uv__req_register(loop, req)等同于handle的uv__active_handle_add。uv__req_register在uv__req_init中调用，几乎（~~漏网的暂时没法解释~~ ）每个req在初始化时都调用了uv__req_init。

```cpp
#define uv__req_init(loop, req, typ)                                          \
  do {                                                                        \
    UV_REQ_INIT(req, typ);                                                    \
    uv__req_register(loop, req);                                              \
  }                                                                           \
  while (0)
  
#define uv__req_register(loop, req)                                           \
  do {                                                                        \
    (loop)->active_reqs.count++;                                              \
  }                                                                           \
  while (0)
```
&emsp;&emsp;下图所示是那些函数调用了uv__req_init，由名称我们可以看出来它们是属于哪些req的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191116181900253.png)
&emsp;&emsp;同理，还有uv__req_unregister。
```cpp
#define uv__req_unregister(loop, req)                                         \
  do {                                                                        \
    assert(uv__has_active_reqs(loop));                                        \
    (loop)->active_reqs.count--;                                              \
  }                                                                           \
  while (0)
```

##### closing_handles 
&emsp;&emsp;要关闭的handle会以链表的形式挂在loop->closing_handles上。这个操作通过调用uv__make_close_pending来实现。
```cpp
void uv__make_close_pending(uv_handle_t* handle) {
  assert(handle->flags & UV_HANDLE_CLOSING);
  assert(!(handle->flags & UV_HANDLE_CLOSED));
  handle->next_closing = handle->loop->closing_handles;
  handle->loop->closing_handles = handle;
}
```
如果closing_handles不为空，那么还需要进入事件循环，去调用关闭的handle的回调函数。