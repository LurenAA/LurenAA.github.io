---
title: redis基数树rax源码分析(2.5)
date: 2019-08-14 18:40:51
tags: ["rax源码阅读"]
---
##### 点点废话
&emsp;&emsp;最近没有再将rax的源码往下看，rax对于一个新手来说还是体量过大，在尝试自己写写，在写的时候遇到了一些坑，也体会到了rax的一些写法的精妙之处，记录一下。

---
##### 宏定义函数的注意点：
&emsp;&emsp;我定义了这样一个宏定义函数：

```
#define radixNthChild(h, n) \
  (radix_node**)((char*)&h->data + h->size + padding(h->size) + n * sizeof(void*))
```
我这样调用这个函数：

```
radixNthChild(new_cur, new_cur->size - 1)
```
这样一个调用大家觉得有问题吗？嗯，肯定是有问题的，不然我说啥?。

这里，按照我们一般的调用函数的思路，这样一个调用的运行过程是这样的：

 1. 计算出new_cur->size - 1
 2. 带入radixNthChild函数

实际上恰恰相反，**宏定义的处理在预编译时（g++ -E）**，宏定义是将对于的定义替换掉，所以在
预编译后的结果如下：
```
# 363 "radix_tree.c"
    memcpy((radix_node**)((char*)&new_cur->data + new_cur->size + ((sizeof(void*) - (sizeof(radix_node) + new_cur->size) % sizeof(void*)) & (sizeof(void*) - 1)) + new_cur->size - 1 * sizeof(void*)), &keyOne, sizeof(void*));
```
可以看到是 ： + new_cur->size - 1 * sizeof(void*)
而不是我所想的： + （new_cur->size - 1） * sizeof(void*)

**可以得出其过程其实是：**

 1. 函数宏定义替换
 2. 运行时计算

 **结论： 在宏定义函数调用时注意括号的问题**，不加括号可能会~~由于运算符优先级而~~ 导致表达式意义与我们想的有出入?


---
##### 地址运算注意点
先给出这样一个结构体：

```
struct test {
	void* a, *b, *c;
};
```

    int main(void) {
	cout << sizeof(test) << endl;
	test* p = new test;
	fprintf(stdout, "%p:%p:%p:%p\n", p, p + 1, (char*)p + 1, (int*)p+1);

	return 0;
	}

&emsp;&emsp;在这样一个测试代码中，大家觉得p + 1, (char*)p + 1, (int*)p+1这三个结果，相对于p的数值相差多少呢？
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190814211800282.png)
&emsp;&emsp;运行结果是这样的。类型与地址的运算是有着密切关系的。

 - p + 1是一个默认情况， 这时1的意义是一个p的地址宽度
 - (char*)p + 1，p被解释为char类型指针，指向的地址被解释为char，于是1就是一个char的地址宽度。

**总结：** 在计算地址时，要注意运算符左边值的类型。你加上的1可能并不是一个字节的大小。

>这是我边看rax边实现的一个小练习，欢迎大家指教：https://github.com/LurenAA/radix_tree ，好想要个star，求求了，兄弟萌:kissing_heart:
