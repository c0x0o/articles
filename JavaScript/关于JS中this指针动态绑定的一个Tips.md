# 关于JS中`this`指针动态绑定的一个Tips

## 概述

`this`的基础知识就不在赘述，具体细节可以参考[阮一峰老师的文章](http://www.ruanyifeng.com/blog/2010/04/using_this_keyword_in_javascript.html)，说的非常仔细。但是今天遇到的问题主要涉及`this`指针的动态性。具体情况是这样的，我先在项目里写了这样一段代码：

```javascript
function StateCenter() {
  var self = this;
  var findEntry = self.findEntry;
  this.__pathEntry__ = {}

  window.onpopstate = function(e) {

    if (entry = findEntry(self.__prev__.url)) {
      // ....
    }

  }
}
StateCenter.prototype.findEntry = function(path, needCreate) {
  // ...
  // error line
  var entry = this.__pathEntry__;
  // ...
}
window.stateCenter = new StateCenter();
```

可以看到，我先在`StateCenter`类的构造函数中用`findEntry`变量引用了自己的一个原型方法（目的其实只是为了简写）。本以为这样可以万事大吉，但是实际上，这段代码会在`errorline`的处发生错误。原因是由于此处的`this`指针指向了`window`而不是期望中的`window.stateCenter`。

## 原因分析

我们知道：

1. 函数中`this`指针的指向是根据其执行环境动态绑定，而不是在声明时绑定。
2. 多个对象嵌套时，`this`指向最近的对象，例如`window.stateCenter.findEntry()`，`this`指向`stateCenter`而不是`window`
3. 在调用时没有指定附属的对象时，`this`总是指向`window`

于是再来看上面的错误代码，错误原因就非常明显了——*我们在调用`findEntry`时实际上并没有为其指定附属对象，因此该函数变成了一个孤立函数，因此动态绑定的`this`被指向了`window`对象*

## 修复问题

我们在调用使用了`this`指针的函数时应当时刻注意其运行环境（即`this`指针的指向）。为了给`findEntry`函数绑定正确的函数，我们决定使用`call`函数，于是这样修改代码：

```javascript
function StateCenter() {
  var self = this;
  var findEntry = self.findEntry;
  this.__pathEntry__ = {}

  window.onpopstate = function(e) {

    // fix 'this' problem
    if (entry = findEntry.call(self, self.__prev__.url)) {
      // ....
    }

  }
}
StateCenter.prototype.findEntry = function(path, needCreate) {
  // ...
  // error line
  var entry = this.__pathEntry__;
  // ...
}
window.stateCenter = new StateCenter();
```

