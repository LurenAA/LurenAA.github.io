---
title: libuv源码分析（6）uv_queue_work
date: 2019-11-26 20:06:58
tags: ["libuv"]
description: 在使用libuv的过程中，我们难免遇见的一个问题是，有一些库没有异步、只能同步运行，这种情况该怎么办呢？如何在libuv中添加一个同步的任务（比如mysql查询）
---
#### 🏤问题来由
&emsp;&emsp;在使用libuv的过程中，我们难免遇见的一个问题是，有一些库没有异步、只能同步运行，这种情况该怎么办呢？比如mysql-connector-cpp。

&emsp;&emsp;首先要说的是，直接在回调函数中执行mysql-connector-cpp这种会阻塞的操作是不符合Libuv的reactor模式的。
```c
void handle_json_lab(std::shared_ptr<smpHttp::HttpRequest> req,std::shared_ptr<smpHttp::HttpResponse> res) 
{
  try{
    Session mq = cli.getSession();
    auto sqlres = mq.sql("Select content FROM labimformation where type = 'labIntroduction'").execute();
    ...
```
&emsp;&emsp;**上面这样便是错误的案例**。我在写[这个项目](https://github.com/LurenAA/smpHttp)时，之前就采用了这样的错误做法。<br>
&emsp;&emsp;我的这个项目是个http后台，我在接受到POST请求，直接在回调函数中执行mysql操作，这时**整个主线程就阻塞住了<sup>1</sup>**，而这就意味着我的http后台不再能接受任何请求，只能等待mysql操作完成后，回调函数返回。而这个mysql的操作耗时一般在3s以上，这对我这个Http后台来说是毁灭性的打击。。。。

<font size=2 color=#483D8B>
1：用户的回调函数是在work->done函数的最后执行的，而work->done是在主线程uv_run中的is_poll中唤醒loop->wq_async后执行的,在work->done函数中阻塞意味着在主线程阻塞住了，uv_run中的事件循环卡住，不再能接受request（这部分不清楚可以去看我的libuv源码分析文章）
</font>

#### 🌆解决办法
&emsp;&emsp;在[手册Thread pool work scheduling](http://docs.libuv.org/en/v1.x/threadpool.html)中为我们这样的需求提供了这样一个函数：<br>**uv_queue_work(uv_loop_t* loop, uv_work_t* req, uv_work_cb work_cb, uv_after_work_cb after_work_cb)**。<br>

&emsp;&emsp;这个函数就是上面我们问题的解决办法。但是要注意的是uv_async_t不可以替代这个。虽然都是执行用户的函数。async是让用户函数直接被**主线程**在uv_run中运行，而uv_queue_work是将work_cb提交给**子线程**执行，完成后通知主线程，主线程在uv_run中执行after_work_cb。

&emsp;&emsp;总结下来就是：uv_async_t用来执行不阻塞的任务，uv_queue_work执行要阻塞的任务（考虑到线程切换的消耗一般不用来执行不阻塞的任务）

#### 🉐看看源码
&emsp;&emsp;这一部分可以结合这[我的这篇文章-libuv源码分析（5）uv_fs_*](https://lurenaa.github.io/2019/11/26/libuv5/)来看。可以作为佐证，libuv中对着这类没有自带异步版本的阻塞操作的处理是一样的：让子线程去执行这个任务，避免阻塞主线程的事件循环，完成后子线程通知主线程。
<br>
uv_queue_work源码：
```c
int uv_queue_work(uv_loop_t* loop,
                  uv_work_t* req,
                  uv_work_cb work_cb,
                  uv_after_work_cb after_work_cb) {
  if (work_cb == NULL)
    return UV_EINVAL;

  uv__req_init(loop, req, UV_WORK);
  req->loop = loop;
  req->work_cb = work_cb;
  req->after_work_cb = after_work_cb;
  uv__work_submit(loop,
                  &req->work_req,
                  UV__WORK_CPU,
                  uv__queue_work,
                  uv__queue_done);
  return 0;
}
```
再结合[我的这篇文章-libuv源码分析（5）uv_fs_*](https://lurenaa.github.io/2019/11/26/libuv5/)中uv_fs_*函数的源码，这些操作可以总结成以下代码：
```c
UV_REQ_INIT(req, typ);          //初始化基类uv_req_t                               
uv__req_register(loop, req);   //添加loop中request的计数，避免uv_run中uv__loop_alive返回0，使得主线程uv_run退出
...//这里是针对不同类型的操作特有的初始化部分
uv__work_submit(loop,
                &req->work_req,
                UV__WORK_CPU, //操作类型
                uv__queue_work, //要阻塞的操作，在fs中是uv__fs_work
                uv__queue_done); //完成后的回调，在fs中是uv__fs_done
```