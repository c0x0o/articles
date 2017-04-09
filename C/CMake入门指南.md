# CMake入门指南

> 该教程基于[官方的Tutorial](https://cmake.org/cmake-tutorial/)

## 概述

CMake的功能类似于GNU autotools，都是使用程序化的方式，自动为项目生成对应平台下的编译规则文件（以下简称Makefile）——比如windows平台下各种IDE的project文件，或者Linux平台下的Makefile。CMake的运作依赖于`CMakeLists.txt`（注意大小写）这个配置文件，这个（些，一个项目中可能存在多个）文件利用各类宏对生成Makefile的过程进行编程。

## 第一步：编译第一个Hello world程序

### 详细过程

创建一个项目目录并在项目根目录创建两个文件`CMakeLists.txt`和`main.c`。

`main.c`的内容：

```c
#include <stdio.h>
int main(void) {
  printf("Hello from main.c!\n");
  
  return 0;
}
```

`CMakeLists.txt`的内容：

```cmake
PROJECT(HELLO)

CMAKE_MINIMUM_REQUIRED(3.0)

SET(SRC_LIST main.c)
MESSAGE(STATUS "The HELLO_BINARY_DIR is " ${HELLO_BINARY_DIR})
MESSAGE(STATUS "The HELLO_SOURCE_DIR is " ${HELLO_SOURCE_DIR})

ADD_EXECUTABLE(hello ${SRC_LIST})
```

之后在根目录下运行：

```shell
cmake .
```

然后一大堆输出后，可以看到CMake通过检查`CMakeLists.txt`在当前目录下生成了一大堆文件，以及最关键的Makefile文件。之后的过程就跟正常的`make`命令使用方式相同了。

### 一些解释

#### `PROJECT`宏

基本格式：`PROJECT(NAME [CXX][C][Java])`

主要用于指定项目名称和项目语言，这里的项目的名称会影响到`<NAME>_BINARY_DIR`和`<NAME>_SOURCE_DIR`两个CMake变量。

#### `CMAKE_MINIMUM_REQUIRED`宏

基本格式：`CMAKE_MINIMUM_REQUIRED(VERSION)`

用于指定需求的最小的`cmake`版本

#### `SET`宏

基本格式：`SET(VAR VALUE)`

`SET`宏可以声明CMake变量，`VALUE`可以使用字符串、变量拼接的方式。

#### `MESSAGE`宏

基本格式：`MESSAGE(STATUS | SEND_ERROR | FATAL_ERROR VALUE)`

`MESSAGE`用于向控制台输出一条信息。

STATUS: 表明向控制台输出一条普通状态信息

SEND_ERROR: 表明向控制台输出一条普通错误信息

FATAL_ERROR: 表明向控制台输出一条致命错误信息，输出这一信息后，CMake过程将被终止

#### `ADD_EXECUTABLE`宏

基本格式：`ADD_EXECUTABLE(EXECUTABLE_NAME SRC_FILES)`

用于生成一个可执行文件，`SRC_FILES`是一个用空格隔开的源文件列表

#### `<name>_BINARY_DIR`和`<name>_SOURCE_DIR`内置变量

这两个变量分别指定了项目编译目录和源码目录，更通俗一点，前者是你执行CMake命令时的目录，而后者是你的`CMakeLists.txt`所在的目录。`<name>`是你用`PROJECT`指定的项目名称（区分大小写）。

## 第二步：更加工程化——添加子目录

### 详细过程

为了让我们的项目更加专业化，现在我们在根目录下创建`build`和`src`两个目录，前者用于存放编译后的可执行文件，而后者则用于存放我们的源代码文件。

需要专门提出的是，我们需要为每一个需要编译的子目录（即包含源文件的子目录）添加一个`CMakeLists.txt`文件，并使用专门的宏命令将这个子目录添加到工程里面，你可以理解为每一个子目录都是一个需要专门指定编译规则的模块。

在创建好目录后，我们做如下更改：

将`main.c`移动到`src`目录中；

在`src`目录中创建`CMakeLists.txt`文件并添加如下内容：

```cmake
ADD_EXECUTABLE(hello main.c)
```

将根目录下的`CMakeLists.txt`文件修改为：

```cmake
PROJECT(HELLO)

ADD_SUBDIRECTORY(src bin)
```

然后切换到`/build/`目录下，执行：

```shell
cmake ..
make
```

可以看见，在`build`目录下生成了`bin`目录以及一系列中间文件，且在`bin`目录下包含我们需要的`hello`程序。

顺便一提，在执行上述命令时，`HELLO_BINARY_DIR`和`HELLO_SOURCE_DIR`分别为`/build`和`/`。

### 一些解释

#### 为什么会有两个Makefile?

如果把每一个子目录看做一个独立的模块，这个问题就是非常好理解的——因为模块的独立性，使得每一个模块应当可以被独立编译。再比如，某个模块并不是被编译为可执行程序，而是被编译为静态或动态库时，我们不仅可以将其作为可执行程序的一个部分，还可以直接将这个模块通过`make install`安装到系统中。

因此，两个Makefile实际上是不一样的。他们分别进行了不同的编译过程，或者说主Makefile调用了了子模块的Makefile。

#### `ADD_SUBDIRECTORY`宏

基本格式：`ADD_SUBDIRECTORY(SOURCE_DIR [BINARY_DIR] [EXCLUDE_FROM_ALL])`

该宏作用是添加一个子目录，这个子目录下包含了一个定义这个模块编译规则的`CMakeLists.txt`的文件，这个文件用于生成一个可以编译该模块的Makefile。第二个参数是可选的，设置后，CMake将会在`${<NAME>_BINARY_DIR}/<BINARY_DIR>`目录下输出模块的编译文件；若不指定`BINARY_DIR`，其默认值是`SOURCE_DIR`。

如果指定了`EXCLUDE_FROM_ALL`选项，那么这个模块不会在父级模块的编译过程被调用时被编译，而是需要手动切换到这个目录下显示的调用编译过程。例如，这个模块是一个example，那么我们需要后面自行编译这些例子，我们就需要为其指定`EXCLUDE_FROM_ALL`选项。例如：

```cmake
ADD_SUBDIRECTORY(src bin EXCLUDE_FROM_ALL)
```

#### 如何指定可执行文件和库文件的输出目录

我们需要指定两个内置变量`EXECUTABLE_OUTPUT_PATH`和`LIBRARY_OUTPUT_PATH`，例如：

```cmake
SET(EXECUTABLE_OUTPUT_PATH ${<NAME>_BINARY_DIR}/bin)
SET(LIBRARY_OUTPUT_PATH ${<NAME>_BINARY_DIR}/lib)
```

## 第三步：编译静态库和动态库

### 详细过程

本节的目的在于，创建包含`hello`函数的C库文件的动态版本和静态版本。

首先在根目录下创建`lib`目录，并在其下创建`hello.h`和`hello.c`，以及`CMakeLists.txt`

现在目录应该像现在这样：

```shell
|------ build
|------ lib
|        |------- CMakeLists.txt
|        |------- hello.c
|        |------- hello.h
|------ src
|        |------- CMakeLists.txt
|        |------- main.c
|------ CMakeLists.txt
```

修改根目录下的`CMakeLists.txt`：

```cmake
PROJECT(HELLO)

MESSAGE(STATUS "This is BINARY dir "${HELLO_BINARY_DIR})
MESSAGE(STATUS "This is SOURCE dir "${HELLO_SOURCE_DIR})

ADD_SUBDIRECTORY(src bin)
ADD_SUBDIRECTORY(lib lib)
```

在`lib`下的`CMakeLists.txt`文件：

```cmake
ADD_LIBRARY(hello SHARED hello.c)
ADD_LIBRARY(hello_static STATIC hello.c)

SET_TARGET_PROPERTIES(hello_static PROPERTIES
    OUTPUT_NAME "hello")
```

`hello.h`文件内容：

```c
#ifndef _HELLO_H_
#define _HELLO_H_

#include <stdio.h>

void hello(void);

#endif
```

`hello.c`的内容：

```c
#include "hello.h"

void hello(void) {
    printf("hello from hello function!\n");
}
```

之后的使用`cmake`命令在`build`目录下进行外部构建，并运行` make`，可以看到在`build/lib`下生成了我们需要的静态版本和动态版本的`hello`库——`libhello.a`和`libhello.so`。

### 一些解释

#### 使用`gcc`创建静态库和动态库

静态库：

``` shell
gcc -c hello.c
ar -rc libhello.a hello.o
```

动态库：

```shell
gcc -fPIC -shared -o libhello.so hello.c
```

#### 库的命名规则

在UNIX系统中，C库的命名遵循以下规则：

`lib<库的名称>.(a|so).<version>`

在cmake中，只要你指定了库的名称和版本，具体输出名称cmake会帮你处理好。

#### `ADD_LIBRARY`宏

基本格式：`ADD_LIBRARY(LIBNAME [SHARED | STATIC | MODULE] [EXCLUDE_FROM_ALL] SOURCE_LIST)`

这个宏的作用与`ADD_EXECUTABLE`类似，都是让`cmake`输出一个二进制文件，只不过条宏生成的是库文件。

第二个参数指明生成的库文件的类型：

SHARED：生成动态库

STATIC：生成静态库

MODULE：对于使用dyld的系统有效，若不支持，则等同与生成动态库

#### `SET_TARGET_PROPERTIES`宏

基本格式：

```cmake
SET_TARGET_PROPERTIES(TARGET1 TARGET2 ...
PROPERTIES
PROP_NAME1 PROP_VAL
PROP_NAME2 PROP_VAL
...)
```

该宏用于设置“被`ADD_XXXX`类宏添加的二进制目标”的属性。在上例中，由于二进制目标的命名不能相同，但我们又希望静态版本和动态版本使用相同的名称。于是我们修改`hello_static`目标的`OUTPUT_NAME`属性，来达到控制库的输出名的目的。

常用的属性还有：

`VERSION`：库版本号

`SOVERSION`：库API版本号

例如，如下设置：

```cmake
SET_TARGET_PROPERTIES(hello
PROPERTIES
VERSION 1.2
SOVERSION 1)
```

会产生如下三个文件（动态库为例）

```shell
libhello.so.1.2
libhello.so.1->libhello.so.1.2
libhello.so->libhello.so.1
```

## 第三步：安装和使用库

### 详细步骤

我们将会在本节中调用在上一节中编译好的`libhello.so`

首先修改`main.c`函数：

```cmake
#include "hello.h"

int main(void) {
    hello();

    return 0;
}
```

然后，修改跟`main.c`的同级的`CMakeLists.txt`：

```cmake
# 添加头文件目录路径
INCLUDE_DIRECTORIES(${HELLO_SOURCE_DIR}/lib)

# 将一个非标准的共享库路径添加到标准路径
LINK_DIRECTORIES(${HELLO_BINARY_DIR}/lib)

ADD_EXECUTABLE(main main.c)
# libhello.so也可以简写成hello

# 这种写法添加位于标准位置的共享库
TARGET_LINK_LIBRARIES(main libhello.so)
```

之后就可以利用`cmake`命令进行构建了。

> TODO：
>
> 使用INSTALL宏，将头文件和库安装到标准的头文件路径、共享库路径

### 一些解释

#### UNIX系统共享库的标准路径

在UNIX系统中，共享库的路径可以通过`/etc/ld.so.conf`指定，而通常情况下，`/etc/ld.so.conf`又是引用了`/etc/ld.so.conf.d`目录下的所有配置文件内容。我们通过查看这些配置文件，可以了解当前系统中，动态链接库和静态链接库的存放位置。

在UNIX系统中，一般情况下库文件通常存放在以下路径中：

```shell
/usr/lib
/usr/local/lib
/lib
```

头文件通常存放在：

```shell
/usr/include
/usr/local/include
```

####  INCLUDE_DIRECTORIES宏

基本格式：`INCLUDE_DIRECTORIES([AFTER|BEFORE] [SYSTEM] DIR_PATH)`

用于指定头文件所在的目录，对应`gcc`的`-I`参数。

前两个参数可选，用于指定目录是被添加到搜索路径之前还是之后。

#### LINK_DIRECTORIES宏

基本格式：`LINK_DIRECTORIES(DIR1 DIR2 ...)`

用于指定非标准的共享库路径，对应`gcc`的`-L`参数。

#### TARGET_LINK_LIBRARY宏

基本格式：`TARGET_LINK_LIBRARY(TARGET LIBRARY1 LIBRARY2 ...)`

用于为二进制文件添加需要链接的共享库，全称和非全称均可。对应`gcc`的`-l`参数

















