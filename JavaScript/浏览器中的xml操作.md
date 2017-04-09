# 浏览器中的XML操作

## XML概述

作为不同应用和平台之间一种交换数据的方式，XML具有良好的结构性，纯文本的格式确保其可以在不同的硬件平台上被方便的使用和识别。

## 如何在AJAX中获取XML

如何利用AJAX API从服务器获取数据在这里不再赘述。获取XML的方式非常简单，只要服务器发送了标准的XML格式，那么可以按照下述方式获取已经解析好的文档：

    var xhr = new XMLHttpRequest();

    // 发送请求
    xhr.onreadystatechange = function () {
        var xmldoc = null;
        
        if (xhr.readystate == 4 && xhr.status == 200) {
            // xmldoc 是已经解析好的XML DOM
            xmldoc = xhr.responseXML;
        }
    }

# 如何解析XML

方法一：适用于IE5以上的IE浏览器

    var xmlDoc = new ActiveXObject("Microsoft.XMLDOC");
    xmlDoc.async = false;
    
    // 从服务器端加载文档
    xmlDoc.load("xxx.xml");
    // 从字符串解析文档
    xmlDoc.loadXML(str);

方法二：适用于firefox及其他浏览器

    // 创建一个空的文档
    var xmlDoc = document.implementation.createDocument("", "", null);
    xmlDoc.async = false;
    
    // 从服务器端加载文档
    xmlDoc.load("xxx.xml");
    // 从字符串解析文档
    xmlDoc.loadXML(str);

方案三：跨浏览器的实现

    var parser = new DOMParser();
    var xmlDoc = parser.parseFromString(str, "text/html");

# 如何操作XML文档

操作XML DOM的操作几乎与操作HTML DOM，唯一需要注意的几点是在创建注释节点和CDATA节点的时候，与创建一般节点所用的方式不尽相同

创建注释节点：

    var comment = document.createComment("comment string");
    // 将此节点添加进文档之中

由于在XML中，除了标签本身，不能出现< > & 等特殊字符，若出现则必须用实体符加以替换，但是在某些特殊情况下，大量的出现这些字符是不可避免的（比如JS代码），于是CDATA就是为了处理这种情况而生。CDATA中的文本将不会被XML解析器所解析，但是CDATA本身会被算作一个节点。下面是一个CDTA的示例：

    <template>
        <script type="text/javascript">
            <![CDATA[
                var a = 6;
                // some js code
            ]]>
        </script>
    </template>

创建CDTA节点：​    
    var cdata = xmldoc.createCDATASection(cdatastr);
    // 将此节点添加进文档之中
