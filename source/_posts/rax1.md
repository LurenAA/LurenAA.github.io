---
title: redis基数树rax源码分析(1)
date: 2019-08-12 00:11:44
tags: ["rax源码阅读"]
---
> &emsp;&emsp;最近想用libuv写个http服务器，看到了这个开源项目[haywire](https://github.com/haywire/haywire)，在看到第39次提交的时候，作者用基数树来存储不同路由的controller，不过在后续版本中改为了使用hash，不过想来不如正好学学基数树，作者使用的基数树是这个版本[radix_tree](https://github.com/j0sh/radixtree)，这个版本缺少注释，且和一般思路不一样的使用的是二叉树而非N叉树，为了理解方便，我选择了注释较多的[rax](https://github.com/antirez/rax)


---
##### 数据结构
&emsp;&emsp;首先要提到的是rax的数据结构设计：

```
typedef struct raxNode {
    uint32_t iskey:1;     /* Does this node contain a key? */
    uint32_t isnull:1;    /* Associated value is NULL (don't store it). */
    uint32_t iscompr:1;   /* Node is compressed. */
    uint32_t size:29;     /
    unsigned char data[];
} raxNode;
```
这里第一个要说到的点是：你觉得这样一个数据结构的大小是多少？24？ 16？ 还是8？
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190811233343948.png)
&emsp;&emsp;**第一个原因**是位域，也就是结构体中的**冒号：**  ，冒号在这里声明实际需要使用的位数，iskey，isnull，iscompr，size四个一共加起来32位，占4个字节。
&emsp;&emsp;**第二个原因**是data[]占0个字节。unsigned char data[];这样一个结构在这里并不是理解成一个指针8个字节。而是一个**柔性数组**的概念，实现一个可变长度。data[1]占结构体1个字节，data[2]占结构体2个字节.......data[13]占13个字节。数组类型的内存是结构体中直接分配的，而不是像指针一样需要我们后来分配。如下图可见：

```
typedef struct raxNode {
    unsigned char data[13];
} raxNode;

int main(int argc, char *argv[])
{
  printf("%d\n", sizeof(raxNode));

  return 0;
}
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190812000143659.png)

---

##### data[]
&emsp;&emsp;接下来我们还是要谈data，在这里data的意义并不是一个简单的unsigned char数组，它存储的是键值key和radixNode指针两种变量。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190812001425256.png)
图来自：https://my.oschina.net/yunqi/blog/3039132
data的实际使用方式在大多数时候是以**内存地址**的方式进行的。

```
#define raxNodeLastChildPtr(n) ((raxNode**) ( \
    ((char*)(n)) + \
    raxNodeCurrentLength(n) - \
    sizeof(raxNode*) - \
    (((n)->iskey && !(n)->isnull) ? sizeof(void*) : 0) \
))
```
&emsp;&emsp;这是访问最后一个节点的函数（也就是访问图中的A-ptr）。n是一个raxNode*指针，对这个指针指向的地址进行＋操作来得到最后一个节点的地址。

---
##### 节点的表示![在这里插入图片描述](https://img-blog.csdnimg.cn/20190812001723801.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMDE1MDQ4,size_16,color_FFFFFF,t_70)
图来自：https://my.oschina.net/yunqi/blog/3039132
&emsp;&emsp;假设基数树中有“abcd”这个键值的节点。那么它的表示形式是像上图这样的。“abcd”这个节点的value-data存储在图片下半部分的节点处，并且下面一个节点iskey设为1.
&emsp;&emsp;**为什么不是直接只有图片的上半部分，由图片上半部分那个节点将iskey设置为1并且将值存储在其value·data中呢？** 
像这样： **[iskey:1][isnull: 0][iscompr:1][size:4][abcd] [z-ptr ][value-ptr]**
#### 先给出结论： 在rax中一个节点的存在（iskey == 1）是由data中对应的子节点来表示的。
**原因很简单：** 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190812002913974.png)
在这个例子里面，这是一个没有压缩的节点，这一层由a和A两个子节点，如果在当前层次表示，如何分辨你指定的是a还是A？所以用引出子节点来表示。

>这是我边看rax边实现的一个小练习，欢迎大家指教：https://github.com/LurenAA/radix_tree ，好想要个star，求求了，兄弟萌:kissing_heart:

