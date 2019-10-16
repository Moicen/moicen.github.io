---
layout: post
title: Echarts饼图标题居中
abstract: 设置Echarts饼图标题居中
category: technical
permalink: echarts-chrome-legend-align
author: 木逸辰
tags: [tech, echarts, chrome]
---

### {{ page.title }}


效果：

![echarts pie align](/assets/images/2018-07-20-echarts-pie-align.jpeg)

代码：

```js
title: [
    {
        text: "成功率",
        left: '48.7%',
        top: '37%',
        textAlign: 'center',
        textBaseline: 'middle',
        textStyle: {
            color: '#dfdfe2',
            fontWeight: 'normal',
            fontSize: 18
        }
    },
    {
        text: `{hili|${parseFloat(statistics.success).toFixed(2)}}{sub|%}`,
        left: '49%',
        top: '52%',
        textAlign: 'center',
        textBaseline: 'middle',
        textStyle: {
            color: '#1a8efe',
            fontWeight: 'bolder',
            rich: {
                hili: {
                    fontSize: 36,
                    height: 36,
                    verticalAlign: "bottom"
                        },
                        sub: {
                    fontSize: 14,
                    height: 22,
                    verticalAlign: "bottom"
                        }
                }
        }
    }
],
```