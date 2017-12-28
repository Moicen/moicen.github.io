---
layout: post
title: 汇编语言对多维数组元素地址的计算
abstract:
category: technical
permalink: multi-dimension-array-address-compute
author: 木逸辰
tags: [tech, assembly code]
---

### {{ page.title }}

对于一维数组`A[R]`，设起始元素地址为`l`，则其任意元素`A[i]`的地址为`l + S * i`