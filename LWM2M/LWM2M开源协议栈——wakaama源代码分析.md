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

## URI类API

### LWM2M实体的抽象表示

对于LWM2M实体（比如一个支持LWM2M协议的设备），可访问服务被抽象为一个一个对象，每一个对象局有三种层次，分别是：Object，Object-Instance，Resource。举例来说，一个LWM2M实体上包含若干提供不同功能的对象（比如说若干种不同的传感器），而每一种功能有可能由多个对象实例提供（比如多个温度传感器，都提供温度读取的功能），这些对象实例实际所能完成的功能被称为资源。（比如温度传感器提供的数据，摄像机拍摄的影像等）。每一个层次在对应的层级上有着独立的ID，分别称为Object ID，Object Instance ID，Resource ID。OMA定义了一些标准的ID，例如：Object ID中的`LWM2M_SECURITY_OBJECT_ID`为0，这一对象用于为节点间的通信提供安全功能。 而`security object`对象中包含一些标准化的资源，比如，`LWM2M_PUBLIC_KEY_ID`标识了`security object`中的公钥资源。Object Instance ID主要用于唯一标识不同的对象实例，一般来说是在设备启动和对象实例化的时候，动态分配（依次从0增长，每一种对象有着不同的实例ID序列）的。

### 通过URI访问LWM2M实体

在LWM2M中，合法的URI应当像如下格式（*注意，不能以`/`结尾*）：

```shell
/<object id>[/<object instance id>][/<resource id>]

# legal example
/0
/0/0
/0/1
/0/1/1
```

URI分别标识了访问资源的object id、object instance id、resource id，后两个id是可选的。

### 在源代码中体现

```c
#define LWM2M_MAX_ID   ((uint16_t)0xFFFF)

#define LWM2M_URI_FLAG_OBJECT_ID    (uint8_t)0x04
#define LWM2M_URI_FLAG_INSTANCE_ID  (uint8_t)0x02
#define LWM2M_URI_FLAG_RESOURCE_ID  (uint8_t)0x01

#define LWM2M_URI_IS_SET_INSTANCE(uri) (((uri)->flag & LWM2M_URI_FLAG_INSTANCE_ID) != 0)
#define LWM2M_URI_IS_SET_RESOURCE(uri) (((uri)->flag & LWM2M_URI_FLAG_RESOURCE_ID) != 0)

typedef struct
{
    uint8_t     flag;           // indicates which segments are set
    uint16_t    objectId;
    uint16_t    instanceId;
    uint16_t    resourceId;
} lwm2m_uri_t;


#define LWM2M_STRING_ID_MAX_LEN 6

// Parse an URI in LWM2M format and fill the lwm2m_uri_t.
// Return the number of characters read from buffer or 0 in case of error.
// Valid URIs: /1, /1/, /1/2, /1/2/, /1/2/3
// Invalid URIs: /, //, //2, /1//, /1//3, /1/2/3/, /1/2/3/4
int lwm2m_stringToUri(const char * buffer, size_t buffer_len, lwm2m_uri_t * uriP);
```

`lwm2m.h`通过`lwm2m_stringToUri`函数，来将URI字符串转化为`lwm2m_uri_t`结构。我们可以通过两个预定义的宏`LWM2M_URI_IS_SET_INSTANCE`和`LWM2M_URI_IS_SET_RESOURCE`来判断URI中后两个ID是否被设置。

## 常量宏

常量宏定义大约在源代码的135行左右，定义了`CoAP Error Code`、标准对象ID、一部分标准资源的ID。

## LWM2M数据类API

### 概述

LWM2M协议定义一种标准的数据类型`lwm2m_data_t`，用于存储各种协议中可能用到的数据。

### 数据结构和常量

```c
typedef enum
{
    LWM2M_TYPE_UNDEFINED = 0,
    LWM2M_TYPE_OBJECT,
    LWM2M_TYPE_OBJECT_INSTANCE,
    LWM2M_TYPE_MULTIPLE_RESOURCE,

    LWM2M_TYPE_STRING,
    LWM2M_TYPE_OPAQUE,
    LWM2M_TYPE_INTEGER,
    LWM2M_TYPE_FLOAT,
    LWM2M_TYPE_BOOLEAN,

    LWM2M_TYPE_OBJECT_LINK
} lwm2m_data_type_t;

typedef struct _lwm2m_data_t lwm2m_data_t;

struct _lwm2m_data_t
{
    lwm2m_data_type_t type;
    uint16_t    id;
    union
    {
        bool        asBoolean;
        int64_t     asInteger;
        double      asFloat;
        struct
        {
            size_t    length;
            uint8_t * buffer;
        } asBuffer;
        struct
        {
            size_t         count;
            lwm2m_data_t * array;
        } asChildren;
        struct
        {
            uint16_t objectId;
            uint16_t objectInstanceId;
        } asObjLink;
    } value;
};
```

### 数据类型与成员的对应关系

`LWM2M_TYPE_OBJECT`,` LWM2M_TYPE_OBJECT_INSTANCE`, `LWM2M_TYPE_MULTIPLE_RESOURCE`:` value.asChildren`

`LWM2M_TYPE_STRING`, `LWM2M_TYPE_OPAQUE`: `value.asBuffer`

`LWM2M_TYPE_INTEGER`,` LWM2M_TYPE_TIME`: `value.asInteger`

`LWM2M_TYPE_FLOAT`: `value.asFloat`

`LWM2M_TYPE_BOOLEAN`: `value.asBoolean`

### 关于OPAQUE

所谓OPAQUE即是指不透明数据类型，这种数据类型是指对用户完全隐藏了内部细节的数据结构。比如UNIX系统中的`pid_t`就是一种opaque数据类型，虽然它通常被实现为一个`long int`类型，但是系统设计者并不希望用户了解其设计细节，用户只需了解如何使用它即可。在LWM2M中，你可以将opaque理解为约定了特殊结构的buffer，只需要知道它的使用方式即可。

### 数据操作API

```c
typedef enum
{
    LWM2M_CONTENT_TEXT      = 0,        // Also used as undefined
    LWM2M_CONTENT_LINK      = 40,
    LWM2M_CONTENT_OPAQUE    = 42,
    LWM2M_CONTENT_TLV       = 11542,
    LWM2M_CONTENT_JSON      = 11543
} lwm2m_media_type_t;

lwm2m_data_t * lwm2m_data_new(int size);
int lwm2m_data_parse(lwm2m_uri_t * uriP, uint8_t * buffer, size_t bufferLen, lwm2m_media_type_t format, lwm2m_data_t ** dataP);
size_t lwm2m_data_serialize(lwm2m_uri_t * uriP, int size, lwm2m_data_t * dataP, lwm2m_media_type_t * formatP, uint8_t ** bufferP);
void lwm2m_data_free(int size, lwm2m_data_t * dataP);

void lwm2m_data_encode_string(const char * string, lwm2m_data_t * dataP);
void lwm2m_data_encode_nstring(const char * string, size_t length, lwm2m_data_t * dataP);
void lwm2m_data_encode_opaque(uint8_t * buffer, size_t length, lwm2m_data_t * dataP);
void lwm2m_data_encode_int(int64_t value, lwm2m_data_t * dataP);
int lwm2m_data_decode_int(const lwm2m_data_t * dataP, int64_t * valueP);
void lwm2m_data_encode_float(double value, lwm2m_data_t * dataP);
int lwm2m_data_decode_float(const lwm2m_data_t * dataP, double * valueP);
void lwm2m_data_encode_bool(bool value, lwm2m_data_t * dataP);
int lwm2m_data_decode_bool(const lwm2m_data_t * dataP, bool * valueP);
void lwm2m_data_encode_objlink(uint16_t objectId, uint16_t objectInstanceId, lwm2m_data_t * dataP);
void lwm2m_data_encode_instances(lwm2m_data_t * subDataP, size_t count, lwm2m_data_t * dataP);
void lwm2m_data_include(lwm2m_data_t * subDataP, size_t count, lwm2m_data_t * dataP);

#define LWM2M_TLV_HEADER_MAX_LENGTH 6

int lwm2m_decode_TLV(const uint8_t * buffer, size_t buffer_len, lwm2m_data_type_t * oType, uint16_t * oID, size_t * oDataIndex, size_t * oDataLen);
```

#### API说明

| 函数名                    | 函数作用                                   |
| ---------------------- | -------------------------------------- |
| `lwm2m_data_parse`     | 将`buffer`中的数据根据`formatP`转化并`dataP`     |
| `lwm2m_data_serialize` | 将`dataP`中的数据根据`formatP`序列化并填入`bufferP` |
| `encode`类函数            | 将特定数据类型的数据编码并填入`dataP`                 |
| `decode`类函数            | 从`dataP`中以特定数据类型提取数据并填入对应指针位置          |

#### 关于TLV

所谓的TLV，实际上是一种三元组编码格式，为了解决发送二进制数据时，由于协议更新导致的对应字段意义变化的问题。举例来说，有如下结构体规定了一种协议：

```c
struct proto_t {
  unsigned int version;
  unsigned int name[50];
}
```

我们在通信时，可以直接将这个结构体的buffer进行发送，接收端接收到buffer后，将其转化为对应的结构体。

但是，当协议进行更新时：

```c
struct proto_new_t {
  unsigned int version;
  unsigned int age;
  unsigned int name[100];
  char address[100];
}
```

客户和服务器不得不同时修改自身的代码，同时增加对应的分支语句，以便向前兼容旧版本的协议。

TLV格式规定每一个字段由三元组描述：`(type [length] value)`，它具有以下特性：

1. 其中`length`字段是可选的，因为当`type`标识`int`等类型时，`length`是已知的
2. `value`可以嵌套新的三元组

可以看出，TLV格式有效避免了由于字段长度变化带来的字段错位问题，并且极大减少了空间浪费（发送端将不会发送50个字节长但实际只使用了3个字节的`name`）。

但是TLV格式并没有解决新增字段带来的兼容性问题，于是有了TLV的增强版——TTLV。

TTLV由`(tag type [length] value)`描述，新增的`tag`字段描述了当前字段的具体含义，简单来说，`tag`为协议增加了自描述特性：

| tag  | 对应字段名   | 字段描述 |
| ---- | ------- | ---- |
| 0    | version | 协议版本 |
| 1    | age     | 年龄   |
| 2    | name    | 姓名   |
| 3    | address | 地址   |

 在处理协议时，我们可以只处理已知的`tag`，从而带来了超好的向前和向后兼容性。

总的来说，使用TLV格式进行编码的协议，将会具有极好的空间特性、向前兼容性、向后兼容性。

## LWM2M对象API

### 概述

抽象对象和LWM2M实体之间的关系，已经在`URI类API`一节中进行过讨论，这里简单进行一下总结：

LWM2M实体（以下简称“实体”）被抽象为提供若干功能的对象，每一个对象被分为三个层次：分别为LWM2M对象，对象实例，绑定于对应实例的资源。

因此，我们为了定位对应的资源（通过URI），需要提供至少三个参数（还会有一些额外的参数，后文再行讨论）：Object ID，Object Instance ID和Resource ID。所以，通常情况下 ，我们访问一个资源的URI长成这个样子：`/0/1/2`。

但是，定位了资源，我们还需要通过特定的方法（通常对应一个或一组对应的函数）来操作这些资源。LWM2M定义了一系列标准方法来操作这些资源，这些方法包括：`read`、`write`、`discover`、`create`、`delete`、`execute`。对于特定的标准对象，LWM2M协议文档规定了每种方法的响应流程，但是对于自定义的对象，我们可以自行约定这些方法的响应方式，也可以只实现其中的任意种方法。

### 对象结构

```c
typedef struct _lwm2m_object_t lwm2m_object_t;
struct _lwm2m_object_t
{
    struct _lwm2m_object_t * next;           // for internal use only.
    uint16_t       objID;
    lwm2m_list_t * instanceList;
    lwm2m_read_callback_t     readFunc;
    lwm2m_write_callback_t    writeFunc;
    lwm2m_execute_callback_t  executeFunc;
    lwm2m_create_callback_t   createFunc;
    lwm2m_delete_callback_t   deleteFunc;
    lwm2m_discover_callback_t discoverFunc;
    void * userData;
};
```

可以看到，LWM2M在对象结构中包含了很多重要的信息：

1. `objID`标识当前对象的ID，也就是Object ID
2. `instanceList`是对象实例的列表，列表和节点类型参看`辅助类功能API`一节
3. 6个事件回调函数（callback）,在对象收到对应方法请求时被调用
4. `userData`是用户自定义的数据，可以存储任意类型和大小数据的指针，一般是跟该对象密切相关的信息（例如配置信息等）

### 回调函数

```c
typedef uint8_t (*lwm2m_read_callback_t) (uint16_t instanceId, int * numDataP, lwm2m_data_t ** dataArrayP, lwm2m_object_t * objectP);

typedef uint8_t (*lwm2m_discover_callback_t) (uint16_t instanceId, int * numDataP, lwm2m_data_t ** dataArrayP, lwm2m_object_t * objectP);

typedef uint8_t (*lwm2m_write_callback_t) (uint16_t instanceId, int numData, lwm2m_data_t * dataArray, lwm2m_object_t * objectP);

typedef uint8_t (*lwm2m_execute_callback_t) (uint16_t instanceId, uint16_t resourceId, uint8_t * buffer, int length, lwm2m_object_t * objectP);

typedef uint8_t (*lwm2m_create_callback_t) (uint16_t instanceId, int numData, lwm2m_data_t * dataArray, lwm2m_object_t * objectP);

typedef uint8_t (*lwm2m_delete_callback_t) (uint16_t instanceId, lwm2m_object_t * objectP);
```

这些回调函数根据命名，分别作为实体接收到对应的方法的请求后，所触发的动作入口（类似于中断处理器或者事件句柄）。需要注意的是，这里定义的实际上是函数类型，需要将对应的函数声明为上述类型才可以使用，这个涉及到C语言的函数指针，本文不再赘述。

下面分别说明各个参数的含义：

| 参数名称         | 参数类型              | 含义                                       |
| ------------ | ----------------- | ---------------------------------------- |
| `instanceId` | `uint_16_t`       | 触发该次事件对象实例的ID                            |
| `numDataP`   | `int *`           | 指出`dataArrayP`中含有的`lwm2m_data_t`数目，由用户函数填入，返回给发送方 |
| `dataArrayP` | `lwm2m_data_t **` | 包含该次操作返回的数据组成的链表，由用户函数填入，返回给发送方          |
| `objectP`    | `lwm2m_object_t`  | 触发该次事件的Object的引用，由协议栈填入                  |
| `numData`    | `int`             | 指明`dataArray`中包含的`lwm2m_data_t`的数目       |
| `dataArray`  | `lwm2m_data_t *`  | 指向该次事件发生时，接收到的`lwm2m_data_t`数据           |
| `resourceId` | `uint16_t`        | 触发该次事件资源ID                               |
| `buffer`     | `uint8_t`         | 指向该次事件发生时，接收到的普通数据                       |
| `length`     | `int`             | 指明`buffer`的长度                            |

