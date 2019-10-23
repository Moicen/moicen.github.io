---
layout: post
title: Chrome下Echarts设置图例项等间距
abstract: 解决Chrome下Echarts图例项间距不一致问题
category: technical
permalink: echarts-chrome-legend-align
author: 木逸辰
tags: [tech, echarts, chrome]
---

### {{ page.title }}


Chrome下echarts图例项的间距很奇怪，与各项文字长度相关，主要是使用官方的`itemGap`设置无效。
正常效果（Safari及Firefox表现）：

![echarts legend](/assets/images/2018-07-02-echarts-legend.jpeg)

异常效果（Chrome表现）：

![echarts legend chrome](/assets/images/2018-07-02-echarts-legend-chrome.jpeg)

参考[#8275](https://github.com/apache/incubator-echarts/issues/8275)issue。

解决方案，参考[#7117](https://github.com/apache/incubator-echarts/issues/7117)issue，使用`textStyle.padding`可以控制图例中文字与图标的间隔，而且此值可以设为负数：

```
data: data.map(i => ({
    name: i.name || i,
    textStyle: {
        padding: [0, 0, 0, -(i.name || i).split("").length * 4]
    },
    icon: i.icon
}))
```

因不同项的文字长度不一致，而这个间距与各项文字长度相关，于是使用文字字数来计算一个长度，经测试，如上文字(全汉字)字数乘以4的宽度较为合适：

![echarts legend chrome fixed](/assets/images/2018-07-02-echarts-legend-fixed.jpg)