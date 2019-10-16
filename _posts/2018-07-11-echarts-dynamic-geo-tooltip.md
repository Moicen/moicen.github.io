---
layout: post
title: Echarts动态地图气泡
abstract: Echarts动态地图气泡
category: technical
permalink: echarts-dynamic-geo-tooltip
author: 木逸辰
tags: [tech, echarts, geo, dynamic]
---

### {{ page.title }}


先看下效果：

![echarts dyamic geo tooltip](/assets/images/2018-07-11-echarts-dynamic-geo-tooltip.gif)

代码：
```js
// 标签块的坐标偏移量
const BaseOffset = {
  x: -57,
  y: -71
};
// 地图数据定位
export default function locate(data) {
  let res = [];
  for (let i = 0; i < data.length; i++) {
    let item = data[i];
    let city = GeoCityMap[item.routeName];
    if (city) {
      let value = item.dispatchCount;
      // 每位数字计算-7px的水平偏移
      let offset = (value.toString().length - 1) * (-7);
      res.push({
        name: data[i].routeName,
        value: city.concat(value),
        label: {
          position: [BaseOffset.x + offset, BaseOffset.y]
        }
      });
    }
  }
  return res;
};

echarts.registerMap('china', china);
option = {
  legend: {
    show: false,
    data: routes.map(b => b.routeName)
  },
  geo: {
    type: 'map',
    map: 'china',
    zoom: 1.25,
    itemStyle: {
      areaColor: 'rgba(6, 24, 62, 0.3)',
      borderColor: '#0c4166',
      borderWidth: 1,
      shadowColor: 'transparent',
    },
    label: {
      show: false
    },
    emphasis: {
      label: {
        show: false
      },
      itemStyle: {
        areaColor: 'rgba(6, 24, 62, 0.3)',
        borderColor: '#0c4166',
        borderWidth: 1,
        shadowColor: 'transparent',
      }
    }
  },
  series: [{
    type: 'scatter',
    coordinateSystem: "geo",
    data: locate(routes),
    symbol: "image:///images/circle-dot.png",
    symbolSize: 15,
    label: {
      show: true,
      backgroundColor: {
        image: "/images/bubble.png"
      },
      textStyle: {
        color: "#fff",
        fontFamily: "PingFang SC",
        fontSize: 13
      },
      height: 85,
      formatter: ["{title|{b}}", "{content|共分拣{@[2]}件}"].join("\n"),
      rich: {
        title: {
          color: "#fff",
          fontSize: 16,
          fontFamily: "PingFang SC",
          padding: [5, 10, 10, 10]
        },
        content: {
          color: "#fff",
          fontSize: 13,
          fontFamily: "PingFang SC",
          padding: [10, 10, 5, 10]
        }
      }
    },
  }]
};
```