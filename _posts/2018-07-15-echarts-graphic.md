---
layout: post
title: Echarts的`graphic`应用
abstract: Echarts的`graphic`应用示例
category: technical
permalink: echarts-graphic-example
author: 木逸辰
tags: [tech, echarts, graphic]
---

### {{ page.title }}


效果：

![echarts graphic](/assets/images/2018-07-15-echarts-graphic.jpeg)

代码：

```js
const series = [{
  key: "working",
  label: "工作",
  color: "#1a8cfe",
  y: -50
}, {
  key: "charging",
  label: "充电",
  color: "#02c759",
  y: 10
}, {
  key: "standby",
  label: "待机",
  color: "#6f6f6f",
  y: 70
}]

option = {
  legend: legend(series.map(s => ({
    name: s.label,
    icon: "rect"
  }))),
  yAxis: {
    name: "单位: 个",
    nameTextStyle: {
      color: "#b9c1dd",
      fontFamily: "PingFang SC",
      fontSize: 10
    },
    axisTick: {
      show: false
    },
    type: "value",
    axisLine: {
      show: false
    },
    splitLine: {
      show: false
    },
  },
  xAxis: {
    show: false
  },
  grid: {
    left: '10%',
    right: '15%',
    bottom: 0,
    containLabel: true
  },
  graphic: {
    type: 'image',
    left: "10%",
    top: "36%",
    style: {
      image: '/images/robot.png',
      width: 170,
      height: 110
    }
  },
  series: series.map(s => ({
    type: "graph",
    layout: "none",
    name: s.label,
    symbolSize: [150, 45],
    symbol: "image:///images/robot-square.png",
    symbolOffset: [110, s.y],
    itemStyle: {
      color: s.color
    },
    label: {
      show: true,
      formatter: `{t|{a}}:    {v|{c}}`,
      textAlign: "left",
      color: "#fff",
      width: 80,
      rich: {
        t: {
          color: "#fff",
          fontFamily: "PingFang SC",
          fontWeight: "lighter",
          fontSize: 16
        },
        v: {
          fontSize: 28,
          fontFamily: "PingFang SC",
          color: s.color
        }
      }
    },
    data: [{
      name: s.label,
      x: 0,
      y: 0,
      value: states[s.key],
    }],
  }))
}
```