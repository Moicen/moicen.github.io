---
layout: post
title: C指针学习笔记
abstract: C指针学习笔记
category: technical
permalink: pointer-note
author: 木逸辰
tags: [tech, c, pointer, function, array]
---

### {{ page.title }}

指针(pointer)是C语言中的一种数据类型，它最显著的特性在于，编译器会将指针变量所存储的数据看作一个内存地址，这个地址可以用来从内存中读取相应的数据。指针的存在实现了高级语言(java、js、C# etc.)中的引用传值能力。

Pointer arguments enable a function to access and change objects in the function that called it.

By definition, the value of a variable or expression of type array is the address of element zero of the array.

In evaluating a[i], C converts it to `*(a+i)` immediately; the two forms are equivalent.

A pointer is a variable, so pa=a and pa++ are legal. But an array name is not a variable; constructions like a=pa and a++ are illegal.

  C guarantees that zero is never a valid address for data, so a return value of zero can be used to signal an abnormal event, in this case no space.
