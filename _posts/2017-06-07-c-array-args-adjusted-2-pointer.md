---
layout: post
title: C语言array参数被隐式转换为指针
abstract: C语言编译时将array类型的参数自动做了隐式转换，转成了指针
category: technical
permalink: c-array-args-adjusted-to-pointer
author: 木逸辰
tags: [tech, c, pointer]
---

### {{ page.title }}

用C写个`binary search`程序，习惯性在函数里直接取数组长度：

    int binary_search(int list[], int target)
    {
        int size = (int) (sizeof(list) / sizeof(int));
        int low = 0, high = size, mid;
        while(low <= high){
            mid = (low + high) / 2;
            if(list[mid] == target) return mid;
            if(list[mid] > target) high = mid - 1;
            else low = mid + 1;
        }
        return -1;
    }

编译时弹出`warning`：

![binary search warning](/assets/images/2017-06-07-c-array-args-2-pointer-1.png)

运行结果显然也不对。搜索了下，发现是C语言对`array`类型参数做了隐式转换：

![binary search trans](/assets/images/2017-06-07-c-array-args-2-pointer-2.png)

所以在函数中对`array`参数求`sizeof`，就是计算的对应指针的`size`，计算对象变了，所以结果也变了。
然后就好奇了，对一个指针计算`sizeof`会得出多少？于是分别在`search`函数外部和内部计算了一下`sizeof`，针对同一个int数组：


    #include <stdio.h>

    int binary_search(int[], int, int);

    int main()
    {
        int my_list[] = {1, 3, 5, 7, 9, 11, 13, 15, 17, 19};
        int size = (int) (sizeof(my_list) / sizeof(int));
        printf("sizeof int: %lu\n", sizeof(int));
        printf("sizeof int[10]: %lu\n", sizeof my_list);
        binary_search(my_list, size, 3);
        }

    int binary_search(int list[], int size, int target)
    {
        printf("sizeof pointer to int[10]: %lu\n", sizeof list);
        return -1;
    }

结果：

![binary search sizeof](/assets/images/2017-06-07-c-array-args-2-pointer-3.png)

第一个`40`很好理解，不过`pointer`的`size`为`8`感觉很奇怪，搜了一下，据说一般会返回跟`sizeof(int)`一致的结果，那么为啥这儿`int`返回`4`，`pointer`返回`8`？于是又到处找，翻了许久的文档，终于找到一个说明[64-bit data models](https://en.wikipedia.org/wiki/64-bit_computing#64-bit_data_models)：

>In 32-bit programs, pointers and data types such as integers generally have the same length. This is not necessarily true on 64-bit machines.[37][38][39] Mixing data types in programming languages such as C and its descendants such as C++ and Objective-C may thus work on 32-bit implementations but not on 64-bit implementations.

>In many programming environments for C and C-derived languages on 64-bit machines, int variables are still 32 bits wide, but long integers and pointers are 64 bits wide. These are described as having an LP64 data model. Another alternative is the ILP64 data model in which all three data types are 64 bits wide, and even SILP64 where short integers are also 64 bits wide.[40] However, in most cases the modifications required are relatively minor and straightforward, and many well-written programs can simply be recompiled for the new environment with no changes. Another alternative is the LLP64 model, which maintains compatibility with 32-bit code by leaving both int and long as 32-bit. LL refers to the long long integer type, which is at least 64 bits on all platforms, including 32-bit environments.

按这里所说，64位机器上，`sizeof(int)`理论上应该返回`8`，`sizeof(pointer)`也是如此，但实际是现在大部分64位机器上`sizof(int)`返回的值依旧与32位机器一致，而`sizeof(pointer)`返回的是按照64位计算的结果。

到此为止，再继续深入的已经不太看得懂了，待以后慢慢学习。
