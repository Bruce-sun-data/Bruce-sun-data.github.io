---
title: 在echarts中利用百度地图api绘制国外地图
date: 2021-04-29 20:06:43
tag: 前端,echarts,百度地图，添加图层
typora-root-url: ..
---



由于我最近要利用High-D数据集做一个数据可视化的大屏，所以下面来看看如何利用echarts+百度地图api对国外地图进行简单的绘制。本文利用了jQuery，不会的同学可以先简单了解一下。

## 申请百度地图的JavaScript API

[百度地图JavaScript API](http://lbsyun.baidu.com/index.php?title=jspopular3.0)是一套由JavaScript语言编写的应用程序接口，可帮助您在网站中构建功能丰富、交互性强的地图应用，支持PC端和移动端基于浏览器的地图应用开发，且支持HTML5特性的地图开发。本项目使用的是v3.0版本。

申请API的具体流程如下图

<img src="/images/在echarts中利用百度地图绘制国外地图/image-20210429214642909.png" alt="image-20210429214642909" style="zoom:50%;" />

其中，需要注意的是在第三步获取服务密钥的时候，应用类型选择浏览器端，申请之后就可以在控制台看到了。

<img src="/images/在echarts中利用百度地图绘制国外地图/image-20210429215213204.png" alt="image-20210429215213204" style="zoom:50%;" />

## Echarts相关资源下载

之后是在echarts的[官方github](https://github.com/apache/echarts/tree/master/dist)中下载相应的js文件，在html中引入的js文件从打包后的`dist`文件夹中拿：`dist/echarts.js`和`dist/extension/bmap.js`，之后再把jquery文件导入到对应的html文件中。

## Echarts官方实例的应用

其中，echarts官网中的[全国主要城市空气质量图](https://echarts.apache.org/examples/zh/editor.html?c=effectScatter-bmap)是利用bmap实现的。html文件的代码如下。

```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <style type="text/css">
        body {
            margin: 0;
        }
        #main {
            height: 100%;
        }
    </style>
</head>
<body>
<div id="main" style="width: 100%;height:800px;"></div>
<script src="./js/echarts.js"></script>
<script src="./js/bmap.js"></script>
<script type="text/javascript" src="http://api.map.baidu.com/api?v=3.0&ak=yourapk"></script>
<script src="./js/mymap.js"></script>
</body>
</html>
```

其中，`mymap.js`文件是自己的新建文件，也就是将上面链接中的完整代码复制过来的。去掉`import * as echarts from 'echarts';`这一行。之后就可以在本地运行，进行查看。如下图所示。

<img src="/images/在echarts中利用百度地图绘制国外地图/image-20210430000527340.png" alt="image-20210430000527340" style="zoom:40%;" />

## 带入到实际实例中

本次项目使用的是德国的地图。

其实百度地图api还是很友善的，因为不管你的需求是国内还是国外它都画的挺详细的（至少在我需求的领域是这个样子）。所以当你想要定位到别的国家的时候，直接修改bmap的center属性就好了，不需要再加载其他国家的地图包之类的了。但是通常其他国家会比中国小，所以有的时候也需要修改地图的缩放程度。自己想要的的经纬度可以使用[百度地图的坐标拾取器 ](http://api.map.baidu.com/lbsapi/getpoint/index.html)获得。在自己代码中找到下面的地方进行修改。

```js
        center: [9.967067, 53.578978],
        zoom: 7,
```

可以看到德国的轮廓被白实线勾勒出来。如下图所示

![image-20210511163319564](/images/在echarts中利用百度地图绘制国外地图/image-20210511163319564.png)

如何去除左下角的百度logo和上面居中的题目。左下角的logo需要添加css内容

```css
.BMap_cpyCtrl {
            display: none;
        }
.anchorBL {
        display: none;
        }
```

标题就删除option中的title，即删掉如下代码。

```
title: {
        text: '全国主要城市空气质量 - 百度地图',
        subtext: 'data from PM25.in',
        sublink: 'http://www.pm25.in',
        left: 'center'
    },
```

地图的样式并不是我们想要的，所以需要对地图的具体样式进行修改，可以使用百度提供的[百度地图个性化编辑器 (baidu.com)](https://lbsyun.baidu.com/customv2/editor/37ac49dc514ca193f95359ab99c72aee)对地图进行个性化定制。根据自己的需求定制后点击编辑json，可以获得此地图样式对应的代码。之后把代码复制到`mymap.js`中的`styleJson`就可以看到内容发生了改变。下面是我的样式对应的代码和结果。并且为了显示具体的城市和街道名称，需要将zoom调至15。

```js
mapStyle: {
            styleJson: [{
                "featureType": "subway",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "off"
                }
            }, {
                "featureType": "highway",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "highway",
                "elementType": "geometry.fill",
                "stylers": {
                    "color": "#fa0404ff"
                }
            }, {
                "featureType": "land",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "color": "#091220ff"
                }
            }, {
                "featureType": "water",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "color": "#113549ff"
                }
            }, {
                "featureType": "green",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "color": "#0e1b30ff"
                }
            }, {
                "featureType": "building",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "building",
                "elementType": "geometry.fill",
                "stylers": {
                    "color": "#ffffffb3"
                }
            }, {
                "featureType": "building",
                "elementType": "geometry.stroke",
                "stylers": {
                    "color": "#dadadab3"
                }
            }, {
                "featureType": "subwaystation",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "color": "#b15454B2"
                }
            }, {
                "featureType": "education",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "color": "#e4f1f1ff"
                }
            }, {
                "featureType": "medical",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "color": "#f0dedeff"
                }
            }, {
                "featureType": "scenicspots",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "color": "#e2efe5ff"
                }
            }, {
                "featureType": "highway",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "weight": "4"
                }
            }, {
                "featureType": "highway",
                "elementType": "geometry.fill",
                "stylers": {
                    "color": "#f7c54dff"
                }
            }, {
                "featureType": "highway",
                "elementType": "geometry.stroke",
                "stylers": {
                    "color": "#fed669ff"
                }
            }, {
                "featureType": "highway",
                "elementType": "labels",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "highway",
                "elementType": "labels.text.fill",
                "stylers": {
                    "color": "#8f5a33ff"
                }
            }, {
                "featureType": "highway",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "highway",
                "elementType": "labels.icon",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "arterial",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "weight": "2"
                }
            }, {
                "featureType": "arterial",
                "elementType": "geometry.fill",
                "stylers": {
                    "color": "#d8d8d8ff"
                }
            }, {
                "featureType": "arterial",
                "elementType": "geometry.stroke",
                "stylers": {
                    "color": "#ffeebbff"
                }
            }, {
                "featureType": "arterial",
                "elementType": "labels",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "arterial",
                "elementType": "labels.text.fill",
                "stylers": {
                    "color": "#525355ff"
                }
            }, {
                "featureType": "arterial",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "local",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "weight": "1"
                }
            }, {
                "featureType": "local",
                "elementType": "geometry.fill",
                "stylers": {
                    "color": "#d8d8d8ff"
                }
            }, {
                "featureType": "local",
                "elementType": "geometry.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "local",
                "elementType": "labels",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "local",
                "elementType": "labels.text.fill",
                "stylers": {
                    "color": "#979c9aff"
                }
            }, {
                "featureType": "local",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "railway",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "weight": "1"
                }
            }, {
                "featureType": "railway",
                "elementType": "geometry.fill",
                "stylers": {
                    "color": "#123c52ff"
                }
            }, {
                "featureType": "railway",
                "elementType": "geometry.stroke",
                "stylers": {
                    "color": "#12223dff"
                }
            }, {
                "featureType": "subway",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on",
                    "weight": "1"
                }
            }, {
                "featureType": "subway",
                "elementType": "geometry.fill",
                "stylers": {
                    "color": "#d8d8d8ff"
                }
            }, {
                "featureType": "subway",
                "elementType": "geometry.stroke",
                "stylers": {
                    "color": "#ffffff00"
                }
            }, {
                "featureType": "subway",
                "elementType": "labels",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "subway",
                "elementType": "labels.text.fill",
                "stylers": {
                    "color": "#979c9aff"
                }
            }, {
                "featureType": "subway",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "continent",
                "elementType": "labels",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "continent",
                "elementType": "labels.icon",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "continent",
                "elementType": "labels.text.fill",
                "stylers": {
                    "color": "#333333ff"
                }
            }, {
                "featureType": "continent",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "city",
                "elementType": "labels.icon",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "city",
                "elementType": "labels",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "city",
                "elementType": "labels.text.fill",
                "stylers": {
                    "color": "#454d50ff"
                }
            }, {
                "featureType": "city",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "town",
                "elementType": "labels.icon",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "town",
                "elementType": "labels",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "town",
                "elementType": "labels.text.fill",
                "stylers": {
                    "color": "#454d50ff"
                }
            }, {
                "featureType": "town",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "road",
                "elementType": "geometry.fill",
                "stylers": {
                    "color": "#12223dff"
                }
            }, {
                "featureType": "poilabel",
                "elementType": "labels",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "districtlabel",
                "elementType": "labels",
                "stylers": {
                    "visibility": "off"
                }
            }, {
                "featureType": "road",
                "elementType": "geometry",
                "stylers": {
                    "visibility": "on"
                }
            }, {
                "featureType": "road",
                "elementType": "labels",
                "stylers": {
                    "visibility": "off"
                }
            }, {
                "featureType": "road",
                "elementType": "geometry.stroke",
                "stylers": {
                    "color": "#ffffff00"
                }
            }, {
                "featureType": "district",
                "elementType": "labels",
                "stylers": {
                    "visibility": "off"
                }
            }, {
                "featureType": "poilabel",
                "elementType": "labels.icon",
                "stylers": {
                    "visibility": "off"
                }
            }, {
                "featureType": "poilabel",
                "elementType": "labels.text.fill",
                "stylers": {
                    "color": "#2dc4bbff"
                }
            }, {
                "featureType": "poilabel",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffff00"
                }
            }, {
                "featureType": "manmade",
                "elementType": "geometry",
                "stylers": {
                    "color": "#12223dff"
                }
            }, {
                "featureType": "districtlabel",
                "elementType": "labels.text.stroke",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "entertainment",
                "elementType": "geometry",
                "stylers": {
                    "color": "#ffffffff"
                }
            }, {
                "featureType": "shopping",
                "elementType": "geometry",
                "stylers": {
                    "color": "#12223dff"
                }
            }]
        }
```

![image-20210511170026972](/images/在echarts中利用百度地图绘制国外地图/image-20210511170026972.png)

## 自定义绘制层

在地图中做出类似百度地图那种路线中有绿有黄有红的感觉。代表路段不同的拥堵程度，就需要使用百度地图api中的[JavaScript API - 自定义绘制层 | 百度地图API SDK (baidu.com)](https://lbsyun.baidu.com/index.php?title=jspopular3.0/guide/drawlayer)介绍挺详细的，但是还是会有一些问题。所以在这里还是进行简单的介绍一下。

首先为了确定公路，需要获得公路的坐标。本项目打算画六段路，所以获取了七个坐标。再mymap.js中创建变量。

```js
let allpoints = {
    '汉堡':[
        new BMap.Point(9.959032, 53.574087),
        new BMap.Point(9.961734, 53.577172),
        new BMap.Point(9.962316, 53.578091),
        new BMap.Point(9.964544, 53.580444),
        new BMap.Point(9.966556, 53.582198),
        new BMap.Point(9.968137, 53.583753),
        new BMap.Point(9.970868, 53.584396),
    ],
}
```

其中`BMap.Point`是一个地理坐标点 ，具体的操作可以看百度api对应的文档，[百度地图JSAPI 2.0类参考 (baidu.com)](https://lbsyun.baidu.com/cms/jsapi/reference/jsapi_reference.html#a1b0)下面是通过一段线将这七个点连起来。

由于本项目是在echarts中使用的百度地图api，所以需要通过echarts调用。具体代码如下

```js
let state_name='汉堡'
//获得bmap
var bmap = myChart.getModel().getComponent('bmap').getBMap();
//    首先清空覆盖物
var allOverlay = bmap.getOverlays();
for (var i = 0; i < allOverlay.length -1; i++){
    bmap.removeOverlay(allOverlay[i]);
}
//添加折线,并进行位置的修改
//    绿色的折线
var curve = new BMap.Polyline(allPoints[state_name], {strokeColor:"#1aff00",strokeWeight:8, strokeOpacity:1});
bmap.addOverlay(curve);
var curve_red = new BMap.Polyline([allPoints[state_name][3], allPoints[state_name][4]], {strokeColor:"#ff0000",strokeWeight:8, strokeOpacity:1});
bmap.addOverlay(curve_red);
//黄色的折线
var curve_yellow = new BMap.Polyline([ allPoints[state_name][1],allPoints[state_name][2]], {strokeColor:"#ffb700",strokeWeight:8, strokeOpacity:1});
bmap.addOverlay(curve_yellow);
// 绘制中心点
var p = new BMap.Marker(centers[state_name]);
bmap.addOverlay(p);
```

由于主要是对图层进行添加，所以还应**删除bmap中的series的内容**。最后的结果如下图所示。

![image-20210511202950352](/images/在echarts中利用百度地图绘制国外地图/image-20210511202950352.png)

`mymap.js`的源码如下。

```js
var chartDom = document.getElementById('main');
var myChart = echarts.init(chartDom);
var option;

let allPoints = {
    '汉堡':[
        new BMap.Point(9.959032, 53.574087),
        new BMap.Point(9.961734, 53.577172),
        new BMap.Point(9.962316, 53.578091),
        new BMap.Point(9.964544, 53.580444),
        new BMap.Point(9.966556, 53.582198),
        new BMap.Point(9.968137, 53.583753),
        new BMap.Point(9.970868, 53.584396),
    ],
}


option = {
    tooltip : {
        trigger: 'item'
    },
    bmap: {
        center: [9.967067, 53.578978],
        zoom: 15,
        roam: true,
        mapStyle: {
            styleJson: [
                {
                    "featureType": "subway",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "off"
                    }
                }, {
                    "featureType": "highway",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "land",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "color": "#091220ff"
                    }
                }, {
                    "featureType": "water",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "color": "#113549ff"
                    }
                }, {
                    "featureType": "green",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "color": "#0e1b30ff"
                    }
                }, {
                    "featureType": "building",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "building",
                    "elementType": "geometry.fill",
                    "stylers": {
                        "color": "#ffffffb3"
                    }
                }, {
                    "featureType": "building",
                    "elementType": "geometry.stroke",
                    "stylers": {
                        "color": "#dadadab3"
                    }
                }, {
                    "featureType": "subwaystation",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "off",
                        "color": "#b15454B2"
                    }
                }, {
                    "featureType": "education",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "color": "#e4f1f1ff"
                    }
                }, {
                    "featureType": "medical",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "color": "#f0dedeff"
                    }
                }, {
                    "featureType": "scenicspots",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "color": "#e2efe5ff"
                    }
                }, {
                    "featureType": "highway",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "weight": 4
                    }
                }, {
                    "featureType": "highway",
                    "elementType": "geometry.fill",
                    "stylers": {
                        "color": "#f7c54dff"
                    }
                }, {
                    "featureType": "highway",
                    "elementType": "geometry.stroke",
                    "stylers": {
                        "color": "#fed669ff"
                    }
                }, {
                    "featureType": "highway",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "highway",
                    "elementType": "labels.text.fill",
                    "stylers": {
                        "color": "#8f5a33ff"
                    }
                }, {
                    "featureType": "highway",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "highway",
                    "elementType": "labels.icon",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "arterial",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "weight": 2
                    }
                }, {
                    "featureType": "arterial",
                    "elementType": "geometry.fill",
                    "stylers": {
                        "color": "#d8d8d8ff"
                    }
                }, {
                    "featureType": "arterial",
                    "elementType": "geometry.stroke",
                    "stylers": {
                        "color": "#ffeebbff"
                    }
                }, {
                    "featureType": "arterial",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "arterial",
                    "elementType": "labels.text.fill",
                    "stylers": {
                        "color": "#525355ff"
                    }
                }, {
                    "featureType": "arterial",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "local",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "weight": 1
                    }
                }, {
                    "featureType": "local",
                    "elementType": "geometry.fill",
                    "stylers": {
                        "color": "#d8d8d8ff"
                    }
                }, {
                    "featureType": "local",
                    "elementType": "geometry.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "local",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "local",
                    "elementType": "labels.text.fill",
                    "stylers": {
                        "color": "#979c9aff"
                    }
                }, {
                    "featureType": "local",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "railway",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "weight": 1
                    }
                }, {
                    "featureType": "railway",
                    "elementType": "geometry.fill",
                    "stylers": {
                        "color": "#123c52ff"
                    }
                }, {
                    "featureType": "railway",
                    "elementType": "geometry.stroke",
                    "stylers": {
                        "color": "#12223dff"
                    }
                }, {
                    "featureType": "subway",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on",
                        "weight": 1
                    }
                }, {
                    "featureType": "subway",
                    "elementType": "geometry.fill",
                    "stylers": {
                        "color": "#d8d8d8ff"
                    }
                }, {
                    "featureType": "subway",
                    "elementType": "geometry.stroke",
                    "stylers": {
                        "color": "#ffffff00"
                    }
                }, {
                    "featureType": "subway",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "subway",
                    "elementType": "labels.text.fill",
                    "stylers": {
                        "color": "#979c9aff"
                    }
                }, {
                    "featureType": "subway",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "continent",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "continent",
                    "elementType": "labels.icon",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "continent",
                    "elementType": "labels.text.fill",
                    "stylers": {
                        "color": "#333333ff"
                    }
                }, {
                    "featureType": "continent",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "city",
                    "elementType": "labels.icon",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "city",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "city",
                    "elementType": "labels.text.fill",
                    "stylers": {
                        "color": "#454d50ff"
                    }
                }, {
                    "featureType": "city",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "town",
                    "elementType": "labels.icon",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "town",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "town",
                    "elementType": "labels.text.fill",
                    "stylers": {
                        "color": "#454d50ff"
                    }
                }, {
                    "featureType": "town",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "road",
                    "elementType": "geometry.fill",
                    "stylers": {
                        "color": "#12223dff"
                    }
                }, {
                    "featureType": "poilabel",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "districtlabel",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "off"
                    }
                }, {
                    "featureType": "road",
                    "elementType": "geometry",
                    "stylers": {
                        "visibility": "on"
                    }
                }, {
                    "featureType": "road",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "off"
                    }
                }, {
                    "featureType": "road",
                    "elementType": "geometry.stroke",
                    "stylers": {
                        "color": "#ffffff00"
                    }
                }, {
                    "featureType": "district",
                    "elementType": "labels",
                    "stylers": {
                        "visibility": "off"
                    }
                }, {
                    "featureType": "poilabel",
                    "elementType": "labels.icon",
                    "stylers": {
                        "visibility": "off"
                    }
                }, {
                    "featureType": "poilabel",
                    "elementType": "labels.text.fill",
                    "stylers": {
                        "color": "#2dc4bbff"
                    }
                }, {
                    "featureType": "poilabel",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffff00"
                    }
                }, {
                    "featureType": "manmade",
                    "elementType": "geometry",
                    "stylers": {
                        "color": "#12223dff"
                    }
                }, {
                    "featureType": "districtlabel",
                    "elementType": "labels.text.stroke",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "entertainment",
                    "elementType": "geometry",
                    "stylers": {
                        "color": "#ffffffff"
                    }
                }, {
                    "featureType": "shopping",
                    "elementType": "geometry",
                    "stylers": {
                        "color": "#12223dff"
                    }
                }
            ]
        }
    },
    series :[],
};
option && myChart.setOption(option);
let state_name='汉堡'
//获得bmap
var bmap = myChart.getModel().getComponent('bmap').getBMap();
//    首先清空覆盖物
var allOverlay = bmap.getOverlays();
for (var i = 0; i < allOverlay.length -1; i++){
    bmap.removeOverlay(allOverlay[i]);
}
//添加折线,并进行位置的修改
//    绿色的折线
var curve = new BMap.Polyline(allPoints[state_name], {strokeColor:"#1aff00",strokeWeight:8, strokeOpacity:1});
bmap.addOverlay(curve);
var curve_red = new BMap.Polyline([allPoints[state_name][3], allPoints[state_name][4]], {strokeColor:"#ff0000",strokeWeight:8, strokeOpacity:1});
bmap.addOverlay(curve_red);
//黄色的折线
var curve_yellow = new BMap.Polyline([ allPoints[state_name][1],allPoints[state_name][2]], {strokeColor:"#ffb700",strokeWeight:8, strokeOpacity:1});
bmap.addOverlay(curve_yellow);
// 绘制中心点
var p = new BMap.Marker(centers[state_name]);
bmap.addOverlay(p);

```













