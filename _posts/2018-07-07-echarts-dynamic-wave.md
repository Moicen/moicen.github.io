---
layout: post
title: Echarts动态波浪图
abstract: 动态波浪图
category: technical
permalink: echarts-dynamic-wave
author: 木逸辰
tags: [tech, echarts, wave, dynamic]
---

### {{ page.title }}


先看下效果：

![echarts dynamic wave](/assets/images/2018-07-07-echarts-dynamic-wave.gif)

这个效果需要安装`echarts-liquidfill`。
使用：

```js
option = {
series: [{
  type: 'liquidFill',
  data: [{
    value: value,
    itemStyle: {
        color: new echarts.graphic.LinearGradient(
            0, 0, x2, y2,
            [
                {offset: 1, color: “#38fa8f”}
                {offset: 1, color: “#038048”}
            ]
        )
    }
  }],
    animation:false, /*amination设为false，波浪仍会波动。否则使用渐变色会报错。https://github.com/ecomfe/echarts-liquidfill/issues/50*/
  radius: '100%',
  outline: { show: false },
  backgroundStyle: {
    color: 'rgba(2, 149, 67, 0.2)'
  },
  label: {
    fontSize: 14,
    fontWeight: "lighter",
    formatter: `${value*100}{sub|分}`,
    height: 14,
    color: "#fff",
    rich: {
      sub: {
        fontSize: 8,
        height: 10,
        verticalAlign: "bottom",
      }
    }
  }
}]
};
```