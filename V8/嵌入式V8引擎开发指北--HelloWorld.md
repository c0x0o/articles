# 嵌入式V8引擎开发指北--HelloWorld.md

## 前言

在上一篇文章中，我们介绍了如何从源代码编译和安装嵌入式版本的V8引擎。如果你正确
的生成了静态链接库并且找到了V8头文件的存放位置（`//include`目录下），那么在本篇
中，我们将会使用已经生成好的静态链接库来创建一个`Hello World`程序。该程序可以
解释运行一段*简单*的JS脚本，并且将脚本的运行结果输出到屏幕上，并且我们将通过这
个最简单的示例程序来探索一些V8引擎使用中的基本概念。

## 创建一个C++工程

### 自动化工具

创建一个C++工程的方式有很多，VS的`.sln`文件，XCode的`.xcodeproj`，用CMake的
`CMakeLists.txt`，用Ninja的`build.ninja`等等。在Linux和Unix平台上最基础、最常见
的创建工程方式是使用Makefile和`make`命令。而在本系列中，我们都将使用Makefile作
为工程自动化工具。

Makefile中定义了一系列编译目标和任务，我们可以选择只编译源代码中的某一个模块，
或者通过定义`all`任务来编译默认的全部目标。许多人认为Makefile只能作为C/C++的自
动化工具，但是作为一系列编译流程命令的合集，配置得当的Makefile理论上可以作为任
何语言的自动化工具，比如笔者就曾经使用Makefile来Compile和Lint项目中CSS代码。

P.S. Makefile在C/C++中的作用*类似*与前端中的Gulp，Grunt和Webpack

### 工程目录结构

```
/--
  |--- src             存放源代码，头文件和源代码混合
  |     |-----main.cc  程序入口
  |
  |--- bin             存放编译好的二进制文件
  |--- Makefile
```

### Makefile

假设v8源码位于你的用户主目录下。

```makefile
V8_BASE = $(HOME)/v8
# INCLUDE指向V8头文件所在的目录
INCLUDE = -I. -I$(V8_BASE)/include
# LINKPATH指向v8_monolith库所在的目录
LINKPATH = -L$(V8_BASE)/out/x64.monolith/obj
LIB = -lv8_monolith -lpthread

all: hello_world

hello_world:
	g++ -std=c++11 $(INCLUDE) src/main.cc -o bin/hello_world $(LINKPATH) $(LIB)
```

这个Makefile定义了两个任务，`hello_world`任务编译并生成可执行文件，`all`则是说
明在没有指定要执行的任务时，默认执行`hello_world`。之后，你可以通过使用命令
`make`和`make hello_world`来编译生成可执行文件，可执行文件默认在`bin`目录下。

## Hello World

### 源代码

先祭出源代码，后面再进行仔细分析。在本篇中，我们的代码只有`main.cc`文件：

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <v8.h>
#include <libplatform/libplatform.h>

const char *js = "var str = 'hello' + ' ' + 'world'; str";

int main(int argc, char **argv) {
  v8::V8::InitializeICUDefaultLocation(argv[0]);
  v8::V8::InitializeExternalStartupData(argv[0]);
  std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
  v8::V8::InitializePlatform(platform.get());
  v8::V8::Initialize();

  v8::Isolate::CreateParams create_params;
  create_params.array_buffer_allocator =
      v8::ArrayBuffer::Allocator::NewDefaultAllocator();
  v8::Isolate *isolate = v8::Isolate::New(create_params);

  {
    v8::HandleScope handle_scope(isolate);

    // run the code
    v8::Local<v8::Context> context = v8::Context::New(isolate);
    v8::Context::Scope context_scope(context);

    v8::Local<v8::String> source = v8::String::NewFromUtf8(isolate, js);
    v8::Local<v8::Script> script = v8::Script::Compile(context, source)
                                               .ToLocalChecked();

    v8::Local<v8::Value> result = script->Run(context).ToLocalChecked();
    v8::String::Utf8Value utf8(isolate, result);

    printf("%s\n", *utf8);
  }

EXIT:
  isolate->Dispose();
  v8::V8::Dispose();
  v8::V8::ShutdownPlatform();
  delete create_params.array_buffer_allocator;

  return 0;
}
```

### 代码解析

#### 头文件

真正需要的头文件只有两个：`v8.h`和`libplatform/libplatform.h`。前者定义了V8中
所有的对外API，后者是一个对用户开放的抽象接口，用户可以通过实现该抽象接口来自定
义V8引擎的各种行为，比如内存告急时的行为、时间戳的实现、用户自定义线程池（用于
异步任务的处理）、自定义内存管理策略等等。当然，你也可以直接使用V8默认的
platform实现。Nodejs直接使用了v8引擎的默认Platform实现，又因为它同时使用了libuv，
因此在Nodejs中实质上有两个线程池存在，一个是v8默认的线程池，另一个是libuv事件
循环的线程池。Nodejs的issue中最近都还有相关的提案，建议Nodejs不要使用v8默认的
Platform，而是自行实现以减少额外的开销。

#### 初始化

对应的代码：

```cpp
v8::V8::InitializeICUDefaultLocation(argv[0]);
v8::V8::InitializeExternalStartupData(argv[0]);
std::unique_ptr<v8::Platform> platform = v8::platform::NewDefaultPlatform();
v8::V8::InitializePlatform(platform.get());
v8::V8::Initialize();
```

第一行，初始化`ICU Data`，ICU是一组方便开发者开发国际化应用的服务，具体介绍可以
参看他们的[官方网站](http://userguide.icu-project.org/icudata)。对于简单的V8应
用，我们在启动时可以不提供`icudata.dat`（前一篇文章提到过），否则需要编译生成并
拷贝至可执行文件同级的目录

第二行初始化外部文件，也就是`native_blob.bin`文件，在编译过程中产生。V8引擎中有
一部分内容仍然是使用js实现的，而这个外部文件就是包含了编译后的js的二进制文件。
不过用户可以通过编译选项使得该文件直接被编译进v8中，而且一个V8的开发人员在一个
邮件讨论中也提到他们会在将来移除这个外部文件（2018.7）。

后面三行代码是初始化`v8::platform`和v8引擎，我们选择使用默认的`platform`

### 创建vm实例

对应的代码：

```cpp
v8::Isolate::CreateParams create_params;
create_params.array_buffer_allocator =
  v8::ArrayBuffer::Allocator::NewDefaultAllocator();
v8::Isolate *isolate = v8::Isolate::New(create_params);
```

所谓`isolate`，可以理解为一个v8引擎实例，它拥有独立的堆空间、垃圾回收器实例。
注意`isolate`实例是线程不安全的，虽然它可以在一个生命周期中运行在不同的线程里，
但是不可以被多个线程并发访问。V8的垃圾回收机制非常高效，而且采用很多极具借鉴意
义的优化手段，笔者会另起文探讨。

### 创建JavaScript上下文

对应的代码：

```cpp
v8::HandleScope handle_scope(isolate);

v8::Local<v8::Context> context = v8::Context::New(isolate);
v8::Context::Scope context_scope(context);
```

`Context`对应我们通常意义上的JavaScript运行上下文，比如在浏览器中运行的JS代码可
以访问到`window`，`document`等全局对象，实际上就是在创建`Context`的时候被添加
上去的。`hello world`程序非常简单，因此没有在全局上下文中添加任何额外的对象，
这意味着我们无法在js代码中访问`window`，`document`甚至`require`等对象，前者由
浏览器实现提供，而后者由Nodejs实现。简单来说，此时的上下文是最纯净的JavaScript
上下文，没有出开语言标准以外的任何对象。

我们通过`v8::Context::New(isolate)`在v8内存堆上申请了一个`Context`对象，但是我
们不能像普通C++代码一样直接使用这个对象的指针。这是由于V8的内存管理机制可能会在
内存回收过程中移动一个对象的位置（比如从短周期对象内存区移动到长周期内存区），
这将会导致该对象的原始指针失效。因此我们需要通过`Local`来持有一个堆上的对象，它
会保证即使对象被移动，用户依然可以通过该句柄访问到这个对象。当然，句柄除了
`Local`，还有比如`Persistent`等其它类型，他们的区别主要在于句柄的生命周期，
这里暂且不表。当持有某个对象的所有句柄都被销毁的时候，这个对象才会被垃圾回收器
回收。

`handle_scope`被声明后，其后所有申请的句柄都会被其管理，直到有新的
`handle_scope`被声明（进入到新的scope域）。而一个`handle_scope`被销毁后，所有的
被其管理的句柄都会被销毁，这实际上是一种RAII机制。`handle_scope`具备嵌套的层级
关系。

### 编译并运行脚本

对应的代码：

```cpp
v8::Local<v8::String> source = v8::String::NewFromUtf8(isolate, js);
v8::Local<v8::Script> script = v8::Script::Compile(context, source)
                                           .ToLocalChecked();

v8::Local<v8::Value> result = script->Run(context).ToLocalChecked();

v8::String::Utf8Value utf8(isolate, result);
printf("%s\n", *utf8);
```

第一行，我们在堆上创建一个*JavaScript字符串对象*（不是C++中的字符串），并且将其
初始化为`js`变量中的脚本源代码。

第二行，编译将该字符串编译为脚本，并且注入全局对象（`context`参数），这个过程类
似在JavaScript中执行`eval(script)`。

第三行，运行该脚本并获取返回值。我们的js脚本如下：

```javascript
var str = 'hello' + ' ' + 'world';
str;
```

最后两行是将结果转化为字符串并且打印，期待的结果是`hello world`。关于v8中的JS
对象体系也能衍生出一系列有意思的话题，现在暂且卖个关子。

### 销毁

对应的代码：

```cpp
isolate->Dispose();
v8::V8::Dispose();
v8::V8::ShutdownPlatform();
delete create_params.array_buffer_allocator;
```


## 结语

本篇通过分析一个最简单的`hello world`程序，简单介绍了一个V8引擎开发中的一些基本
概念，在下一篇文章中，我们将着重描述如何将C++对象包装成js对象，并且注入到全局上
下文中提供一系列功能，比如Nodejs提供的`require`，浏览器提供的`document`对象等。
