---
layout: post
title: Echarts动态提示框
abstract: Echarts动态提示框
category: technical
permalink: echarts-dynamic-tooltip
author: 木逸辰
tags: [tech, echarts, tooltip, dynamic]
---

### {{ page.title }}

先看下效果：

![echarts dynamic tooltip](/assets/images/2018-07-05-echarts-dynamic-tooltip.gif)

提示框绘制：

```js
tooltip : {
  backgroundColor: colors[index],
  borderRadius: 3,
  padding: 3,
  trigger: "axis",
  position: point => [point[0] - 23, point[1] - 40],
  hideDelay: 10000,
  textStyle: {
    color: "#fff",
    fontFamily: "PingFang SC",
    fontSize: 12
  },
  axisPointer: {
    lineStyle: {
      color: colors[index],
      opacity: 0.3
    }
  },
  formatter: (params) => {
    if(count++ >= len) {
      index = Number(!index);
      count = 1;
    };
    let spanStyle = "position:absolute;width:0;height:0;font-size:0;overflow:hidden;right:13px;bottom:-16px;border-style:solid;"
      + `border-width:7px; border-color: ${colors[index]} transparent transparent transparent;`;
    return `<div style="position:relative; width: 40px; text-align: center;">
        <span style='${spanStyle}'></span>
        <strong style="font-size: 12px">
          ${params[index].value}
        </strong>
      </div>`
  }
}
```

动态：
```js
let i = 0, s = 0, colors = ["#1aabfe", "#17cf74"];
  timer.trend = setInterval(() => {
    if(i >= data.length) {
      s = Number(!s);
      let opt = {
        tooltip: {
          backgroundColor: colors[s],
          axisPointer: {
            lineStyle: {
              color: colors[s]
            }
          }
        }
      };
      chart.setOption(opt);
      i = 0;
    };
    chart.dispatchAction({ type: 'showTip', seriesIndex: s, dataIndex: i });
    i++;
  }, 1500);
```
