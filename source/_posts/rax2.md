---
title: redis基数树rax源码分析(2)
date: 2019-08-12 18:51:18
tags: ["rax源码阅读"]
---

> 今天我想要说的是rax中的padding这个函数，我查了很多的资料，大家的博客都告诉我们内存对齐提高性能，却没有去分析为什么，是有根据让作者选择这样做？如果只是这样简单的放过，总感觉让人有一丝的遗憾。

---

&emsp;&emsp;
**先把主角拉出来：**
```
#define raxPadding(nodesize) 
((sizeof(void*)-((nodesize+4) % sizeof(void*))) & (sizeof(void*)-1))
```
&emsp;&emsp;首先要说的是raxPadding的作用是：让raxNewNode申请的内存nodesize是8的倍数。

```
raxNode *raxNewNode(size_t children, int datafield) {
    size_t nodesize = sizeof(raxNode)+children+raxPadding(children)+
                      sizeof(raxNode*)*children;
    if (datafield) nodesize += sizeof(void*);
    raxNode *node = rax_malloc(nodesize);
    if (node == NULL) return NULL;
    node->iskey = 0;
    node->isnull = 0;
    node->iscompr = 0;
    node->size = children;
    return node;
}
```
###### 第一个问题：对齐的优势
&emsp;&emsp;这个并不是我想说的重点，这里是大家都谈到的，也就是**经过内存对齐之后，CPU的内存访问速度大大提升**。对于我来说，这个结论感觉还是太模糊，这是一个定性的结论，具体的底层细节对于我们初学者来说倒是没必要去深究。

###### 第二个问题：为什么要这么去做？
&emsp;&emsp;rax的作者这样的做法其实是**参考结构体的做法**。
**举个例子：**
```
struct X
{
    char a;
    int c;
    double b;
}S2;
```
这样一个结构体，它的大小是多少？答案是16。
在c语言的内部，做了这样的内存对齐处理：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190812190523897.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMDE1MDQ4,size_16,color_FFFFFF,t_70)

> 这里转载了[这篇文章](https://www.cnblogs.com/zhoujiayi/p/7872262.html)中的很多资源，大家也可以去看看这篇文章，写的很不错。也有更多例子。


##### 回到rax上来
&emsp;&emsp; 在rax的raxNode这个结构体中，~~因为使用了**柔性数组**，所以在c语言本身是无法帮助我们实现像上面一样的内存对齐的（sizeof(raxNode) == 4,我们申请的内存大小决定了柔性数组的长度，详情请百度柔性数组）~~ ，c语言对于结构体的优化没有包含柔性数组这个部分，所以**我们必须自己来接管这一部分的内存对齐，保证程序的运行效率。**

    typedef struct raxNode {
    uint32_t iskey:1;     /* Does this node contain a key? */
    uint32_t isnull:1;    /* Associated value is NULL (don't store it). */
    uint32_t iscompr:1;   /* Node is compressed. */
    uint32_t size:29;     /* Number of children, or compressed string len. */
    unsigned char data[];
    } raxNode;


>这是我边看rax边实现的一个小练习，欢迎大家指教：https://github.com/LurenAA/radix_tree ，好想要个star，求求了，兄弟萌:kissing_heart: