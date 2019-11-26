---
title: libuv源码分析（3）init_threads
date: 2019-11-18 02:14:58
tags: ["libuv"]
description: 在我们第一次提交io操作时，会有uv_once被调用，来检测是否初始化过线程池，如果没有则立刻初始化线程池。所以说线程池并非一开始在uv_run的时候或者在loop中初始化的，而是在io操作开始前。
---

#### 由来
&emsp;&emsp;在我们第一次提交io操作时，会有uv_once被调用，来检测是否初始化过线程池，如果没有则立刻**初始化线程池**。所以说线程池并非一开始在uv_run的时候或者在loop中初始化的，而是在io操作开始前。
**我以uv_open为例子画一下UML图如下：**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191118023241696.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMDE1MDQ4,size_16,color_FFFFFF,t_70)
在uv_open中先初始化req，然后准备提交work，提交前会调用uv_once检测是否初始化线程池，没有则初始化。
#### init_once
uv_once实现如下：

```cpp
#define UV_ONCE_INIT PTHREAD_ONCE_INIT
static uv_once_t once = UV_ONCE_INIT;

static void init_once(void) {
#ifndef _WIN32
  /* Re-initialize the threadpool after fork.
   * Note that this discards the global mutex and condition as well
   * as the work queue.
   */
  if (pthread_atfork(NULL, NULL, &reset_once))
    abort();
#endif
  init_threads();
}
```
在uv__work_submit中uv_once是这样被调用的：

```cpp
void uv__work_submit(...) {
  uv_once(&once, init_once);
  ...
}

```
&emsp;&emsp;这一部分可以参看TLPI 31.2部分，libuv多做了pthread_atfork的处理。
&emsp;&emsp;pthread_atfork注册reset_once函数，在fork之后重置once，保证在libuv循环中如果你fork了一个进程，如果在那个新的进程中你也启动一个libuv，init_threads()能被调用。

#### init_threads
###### 🐤条件变量
&emsp;&emsp;libuv初始化**条件变量**时，调用自己的uv_cond_init，这个函数只做了一件事情，就是将**条件变量**的时钟设置为相对时间，这一点是值得我们自己写代码时参考的，相对时间不受系统时间的影响。

```cpp
int uv_cond_init(uv_cond_t* cond) {
  ...
  err = pthread_condattr_setclock(&attr, CLOCK_MONOTONIC);
  ...
}

```
###### 🥛互斥锁
&emsp;&emsp;初始化**互斥锁**时，调用uv_mutex_init，在DEBUG时，libuv会将**互斥锁**设置为PTHREAD_MUTEX_ERRORCHECK，这样能自我检测是否为死锁，不过这会消耗性能，所以在运行时设置为默认值。
```cpp
int uv_mutex_init(uv_mutex_t* mutex) {
#if defined(NDEBUG) || !defined(PTHREAD_MUTEX_ERRORCHECK)
  return UV__ERR(pthread_mutex_init(mutex, NULL));
#else
  ...
  if (pthread_mutexattr_settype(&attr, PTHREAD_MUTEX_ERRORCHECK))
    abort();
  ...
}
```

> PTHREAD_MUTEX_ERRORCHECK
This type of mutex provides error checking. A thread attempting to relock this mutex without first unlocking it shall return with an error. A thread attempting to unlock a mutex which another thread has locked shall return with an error. A thread attempting to unlock an unlocked mutex shall return with an error.

###### 🥡信号量
&emsp;&emsp;初始化每个线程时，libuv用**信号量**来保证init_threads函数在初始化完所有线程后退出。

```cpp
if (uv_sem_init(&sem, 0))
    abort();

  for (i = 0; i < nthreads; i++)
    if (uv_thread_create(threads + i, worker, &sem))
      abort();

  for (i = 0; i < nthreads; i++)
    uv_sem_wait(&sem);

  uv_sem_destroy(&sem);
```
在linux下并且glibc版本大于2.21时，uv_sem_init(&sem, 0)和sem_init(&sem, 0)是一样的，没有额外的处理。
线程创建好后，在worker函数中会调用uv_sem_post释放**信号量**。

```cpp
static void worker(void* arg) {
  ...
  uv_sem_post((uv_sem_t*) arg);
  ...
 }
```
###### 🥚uv_thread_create
&emsp;&emsp;uv_thread_create做的事情就是**设置线程的stack大小**，然后创建它。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191118025701932.png)
thread_stack_size函数获取栈大小，有一些是跨平台兼容性的处理。

```cpp
lim.rlim_cur -= lim.rlim_cur % (rlim_t) getpagesize(); 和
if (lim.rlim_cur >= PTHREAD_STACK_MIN)
        return lim.rlim_cur;
```
上面两行的限制是来源于pthread_attr_setstacksize函数，一下是pthread_attr_setstacksize函数man手册的一部分。

> ERRORS
       pthread_attr_setstacksize() can fail with the following error:
EINVAL The stack size is less than PTHREAD_STACK_MIN (16384) bytes.
 On some systems, pthread_attr_setstacksize() can fail with the error EINVAL if stacksize is not a multiple of
       the system page size.

```cpp
static size_t thread_stack_size(void) {
#if defined(__APPLE__) || defined(__linux__)
  struct rlimit lim;

  if (getrlimit(RLIMIT_STACK, &lim))
    abort();

  if (lim.rlim_cur != RLIM_INFINITY) {
    /* pthread_attr_setstacksize() expects page-aligned values. */
    lim.rlim_cur -= lim.rlim_cur % (rlim_t) getpagesize();

    /* Musl's PTHREAD_STACK_MIN is 2 KB on all architectures, which is
     * too small to safely receive signals on.
     *
     * Musl's PTHREAD_STACK_MIN + MINSIGSTKSZ == 8192 on arm64 (which has
     * the largest MINSIGSTKSZ of the architectures that musl supports) so
     * let's use that as a lower bound.
     *
     * We use a hardcoded value because PTHREAD_STACK_MIN + MINSIGSTKSZ
     * is between 28 and 133 KB when compiling against glibc, depending
     * on the architecture.
     */
    if (lim.rlim_cur >= 8192)
      if (lim.rlim_cur >= PTHREAD_STACK_MIN)
        return lim.rlim_cur;
  }
  ...
  return 2 << 20;  /* glibc default. */
#endif
```
😂无趣的是在linux Ubuntus我的环境下测试时，attr的默认stacksize和thread_stack_size函数设置到的是一样的值。下面是我的测试代码：

```cpp
#include <assert.h>
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <uv.h>
#include <string>
#include <iostream>
#include <cstring>
#include <malloc.h>
#include <time.h>
#include <iostream>
#include <sys/time.h>
#include <sys/resource.h>

using namespace std;
void a(void *) {
  cout << 123 << endl;
}

size_t stack_page() {
  rlimit x;
  assert(getrlimit(RLIMIT_STACK, &x) == 0);
  size_t stack_size = x.rlim_cur - x.rlim_cur % getpagesize();
  cout << stack_size << endl;
  if(stack_size > PTHREAD_STACK_MIN) 
    return stack_size;
}

int main() {
  pthread_attr_t attr;
  assert(pthread_attr_init(&attr) == 0);
  size_t stack_size;
  pthread_attr_getstacksize(&attr, &stack_size);
  cout << stack_size << endl;
  stack_size = stack_page();
  pthread_attr_setstacksize(&attr, stack_size);
  pthread_t p1;
  pthread_create(&p1, &attr, (void* (*)(void*))a, nullptr);
  pthread_attr_destroy(&attr);
  return 0;
}
```
