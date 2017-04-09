# 浏览器历史History对象

在浏览器中，历史浏览记录可以通过History对象进行操作。

先介绍几个关于浏览器历史的一个重要概念：历史记录。历史记录代表每一次浏览器级的跳转动作，这种跳转动作会发生在以下三种情况：

1. 用户点击了一个链接
2. 用户触发了回退操作
3. 用户触发了前进操作

这三种情况下在以前通常意味着页面进行了刷新操作（也可能是页内锚点跳转），但是在如今AJAX大行其道的时代，我们经常会遇到在一个页面根据用户行为进行了大量AJAX操作时，一个浏览器级的跳转导致所有AJAX操作前功尽弃的情况。为了解决这个问题，人们提出了将历史记录提升为页面状态的方式，也就是state。

在现代浏览器中，每一次跳转动作将会被归纳为session中（也就是一个浏览器tab中）的一次状态转化，也就是说现在history栈中存储的其实是session的状态(state)，每次跳转、前进、后退将会引起状态栈的push、pop、replace。这会带来的一个巨大的好处——我们可以把页面内的交互行为也作为一次state变化压入或弹出历史栈。我们将可以不再用劫持用户跳转动作(通常是监听事件)的方式来自己实现控制页面内交互流程产生的状态(state)变化，而是通过浏览器API来实现。这种改进为AJAX应用的状态记录提供了浏览器级别的支持。

详细的介绍可以参看[MDN的文档](https://developer.mozilla.org/en-US/docs/Web/API/History)。我们这里只罗列几个重要的属性和方法：

```javascript
// 存储当前session的history对象中的记录(state)条数（包含当前页面）
history.length
// 当前记录栈顶层的状态(state)数据
history.state

// 浏览器级动作(action)，回退
history.back()
// 浏览器级动作(action)，前进
history.forward()
// 向浏览历史中加入一条状态(state)
// state可以是任意一个能被序列化的对象，可以被onpopstate事件的event.state所接收
// title参数被大多数浏览器忽略，通常情况下我们传一个null
// 可选参数URL，用于更新该状态下的浏览器的URL(可以通过window.location.href取到)
history.pushState(state, title[, url])
// 替换状态栈顶端的状态
history.replaceState(state, title[, url])
// 用于监听状态弹出事件
window.onpopstate = function(e) {
  // 通过event.state来取出在pushState函数中传入的state数据
}
```

那么每次我们执行完一次ajax操作后，可以对应的在浏览器的History对象中添加一条状态

```javascript
ajax.get('/path/to/url').success(function(json){
  // excute some callback code here
  
  // json是我们传入的数据，可以被onpopstate事件和history.state取用，url后面添加了#state1
  history.pushState(json, null, '#state1')
});

// 然后可以通过下述两种方式取用state数据
window.onpopstate = function(e) {
  var state = e.state;
}
// 第二种方式只能取到栈顶层的数据
var state = history.state;
```

需要注意的是，popstate事件只能被同一个页面(same document)内的浏览器级跳转(例如锚点)或者`replaceState()`所触发。

有一个小细节需要注意的是，Chrome(>v34)和Safari(>10.0)会在会在页面加载完成时(on page load)触发一次popstate事件，但是Firefox则不会。