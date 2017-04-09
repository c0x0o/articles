# LWM2M开源协议栈——wakaama源代码分析

> 本文主要分析`liblwm2m.h`文件中非DEBUG状态下的代码，其他代码见后续文章

## 概述

wakaama呈现形式并不是一个C的库文件，而是以源代码的形式，直接和项目代码联合编译。编译模式分为：`LWM2M_SERVER_MODE`、`LWM2M_CLIENT_MODE`、`LWM2M_BOOTSTRAP_SERVER_MODE`三种，分别对应此次编译产生的是服务器、客户端还是启动服务器。

分析顺序遵循，`lwm2m.h`文件从上到下的顺序，按照功能划分对函数原型和功能进行分析。

若有特殊情况，如上下代码关联，必须一起介绍的情况，会有特殊说明

## 如何编译

`wakaama`项目采用`cmake`作为项目构建工具，如何引用项目的`CMakeLists.txt`，可以参照在`/example`下的各个项目的`CMakeLists.txt`是如何编写的。需要注意的是，为了不破坏原有的项目结构，推荐采用外部构建的方式，即：

```shell
# 在wakaama之外新建你自己的项目目录，假设为project
cd project
# 注意替换变量为wakaama项目的所在位置，以编译server为例
# 将会在project目录下生成中间文件
cmake ${wakaama_base_dir}/example/server
make
# 产生的二进制文件名，可以参看example/server/CMakeLists.txt的PROJECT指令的参数
./lwm2mserver
```

你自己的项目可以参看例子的写法。

## 内存管理类API

```c
void * lwm2m_malloc(size_t s);
void * lwm2m_free(void * p);
char * lwm2m_strdup(const char * str);
int lwm2m_strncmp(const char * s1, const char * s2, size_t n);
```

没有需要特别说明的函数，跟UNIX标准函数工作方式类似。

第三个函数作用是产生一个能放入`str`的内存空间，并将`str`的内容复制（duplicate）其中。

## 时间类API

```c
time_t lwm2m_gettime(void);
```

返回距离上一次调用该函数时所流逝（elapse）的时间。根据POSIX规范，`time_t`是一个有符号整数。错误时返回负数。

## 辅助功能类API

```c
typedef struct _lwm2m_list_t
{
    struct _lwm2m_list_t * next;
    uint16_t    id;
} lwm2m_list_t;

lwm2m_list_t * lwm2m_list_add(lwm2m_list_t * head, lwm2m_list_t * node);
lwm2m_list_t * lwm2m_list_find(lwm2m_list_t * head, uint16_t id);
lwm2m_list_t * lwm2m_list_remove(lwm2m_list_t * head, uint16_t id, lwm2m_list_t ** nodeP);
uint16_t lwm2m_list_newId(lwm2m_list_t * head);
void lwm2m_list_free(lwm2m_list_t * head);

#define LWM2M_LIST_ADD(H,N) lwm2m_list_add((lwm2m_list_t *)H, (lwm2m_list_t *)N);
#define LWM2M_LIST_RM(H,I,N) lwm2m_list_remove((lwm2m_list_t *)H, I, (lwm2m_list_t **)N);
#define LWM2M_LIST_FIND(H,I) lwm2m_list_find((lwm2m_list_t *)H, I)
#define LWM2M_LIST_FREE(H) lwm2m_list_free((lwm2m_list_t *)H)
```

提供了链表操作及其简化写法的宏。链表操作使用方式顾名思义。

`lwm2m_list_newId`函数返回指定链表内未被使用的最小id。

`lwm2m_list_free`函数工作方式是：仅对每个节点调用`lwm2m_free`。

