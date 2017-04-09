# stix-viz工具调研
## 总体概述

根据目前的调研情况，viz工具的主要实现形式是基于一个被称为nw.js的工具，通过在浏览器环境下执行js代码实现基本功能,实现STIX-VIZ的基本功能。

stix-viz共有三种显示方式：Graph View, Tree View, Timeline View，默认的展示方式是Graph View,此次项目中要剥离的主要逻辑也是Graph VIew。

## [NW.js](http://nwjs.io/)

nw是node-webkit的缩写，本质上是nodejs和chromium的集合体，让用户可以在chromium内核提供的浏览器环境中，依然可以调用nodejs的所有模块以及nw提供的一些特殊API。它本身是一个封装好的环境，需要用户提供的仅仅是一个类npm项目，然后与nw应用本身直接打包在一起。

简单的使用流程可以参考这个[链接](https://www.sitepoint.com/cross-platform-desktop-app-nw-js/)。从这个教程可以看到，其实就是在你的应用项目内，通过npm安装nwjs，然后使用它自带的打包工具对项目进行打包。

## 渲染流程

先获取XML文档，然后解析为json对象，最后根据这个json对象渲染图形

## 应用顶层逻辑StixViz.js

关键函数:
1. `handleFileSelect()`

## 渲染关键STIXRelationshipGraph.js

封装较好，整个文件就是一个类，用于渲染根据XML生成的json对象

## 关键函数调用流程

    // 全局变量
    // view直接管理渲染逻辑，view.display(json) 
    view = new StixGraph();
    viewType = "selectView-graph"
    // STIXJsonGeneration.js
    // displayJson是一个回调函数，调用view进行渲染
    GenerateJsonForFiles(xmlFiles, viewType, displayJson)
    // STIXRelationshipJson.js
    // TopNodeName指xml的文件名
    // TopLevelNodes指已经构造好的json对象
    // TopLevelObj指从xml文档中提取的dom对象
    gatherRelationshipTopLevelObjs(xml, TopLevelObjects)
    processTopLevelObjects(TopLevelObjects, TopLevelNodes)
    createRelationshipJson(jsonRelationshipObj, TopLevelNodes, TopNodeName)
    // 使用view对象进行渲染
    view.display(relationshipJson)