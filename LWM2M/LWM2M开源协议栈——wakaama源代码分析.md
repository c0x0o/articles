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

## LWM2M运行环境

### 概述

所谓运行环境，通常包含了一系列运行时需要的数据和操作。表现在具体的代码实现中，则是一个包含了若干成员变量和函数的结构体。在`wakaama`中，这个运行时环境被实现为一个名为`lwm2m_context_t`的结构体，该结构体抽象了一个正在运行的服务器、客户端或者启动服务器的实体。

### 结构体结构

```c
typedef struct
{
#ifdef LWM2M_CLIENT_MODE
    lwm2m_client_state_t state;
    char *               endpointName;
    char *               msisdn;
    char *               altPath;
    lwm2m_server_t *     bootstrapServerList;
    lwm2m_server_t *     serverList;
    lwm2m_object_t *     objectList;
    lwm2m_observed_t *   observedList;
#endif
#ifdef LWM2M_SERVER_MODE
    lwm2m_client_t *        clientList;
    lwm2m_result_callback_t monitorCallback;
    void *                  monitorUserData;
#endif
#ifdef LWM2M_BOOTSTRAP_SERVER_MODE
    lwm2m_bootstrap_callback_t bootstrapCallback;
    void *                     bootstrapUserData;
#endif
    uint16_t                nextMID;
    lwm2m_transaction_t *   transactionList;
    void *                  userData;
} lwm2m_context_t;
```

我们可以很明显的注意到，`wakaama`通过宏定义将服务器、客户端、启动服务器三种环境均集成在同一个结构体类型中，这就解释了为什么在`wakaama`中编译中，可以同时定义`LWM2M_SERVER_MODE`、`LWM2M_CLIENT_MODE`、`LWM2M_BOOTSTRAP_SERVER_MODE`——同一个实体只要提供了对应的功能，那么我们可以认为它同时具备这三者的功能。下面我们将对其中几个重要的类型进行解析。

### `lwm2m_object_t`	

#### 概述

抽象对象和LWM2M实体之间的关系，已经在`URI类API`一节中进行过讨论，这里简单进行一下总结：

LWM2M实体（以下简称“实体”）被抽象为提供若干功能的对象，每一个对象被分为三个层次：分别为LWM2M对象，对象实例，绑定于对应实例的资源。

因此，我们为了定位对应的资源（通过URI），需要提供至少三个参数（还会有一些额外的参数，后文再行讨论）：Object ID，Object Instance ID和Resource ID。所以，通常情况下 ，我们访问一个资源的URI长成这个样子：`/0/1/2`。

但是，定位了资源，我们还需要通过特定的方法（通常对应一个或一组对应的函数）来操作这些资源。LWM2M定义了一系列标准方法来操作这些资源，这些方法包括：`read`、`write`、`discover`、`create`、`delete`、`execute`。对于特定的标准对象，LWM2M协议文档规定了每种方法的响应流程，但是对于自定义的对象，我们可以自行约定这些方法的响应方式，也可以只实现其中的任意种方法。

总结来说，`lwm2m_object_t`抽象了客户端实体中的资源。

#### 对象结构

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

#### 回调函数

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

### `lwm2m_server_t`

#### 概述

这个结构体描述了*远端*服务器实体，注意，这个结构体应当只被客户端实体用来记录远端服务器的信息。

#### 结构体结构

```c
typedef struct _lwm2m_server_
{
    struct _lwm2m_server_ * next;         // matches lwm2m_list_t::next
    uint16_t                secObjInstID; // matches lwm2m_list_t::id
    uint16_t                shortID;      // servers short ID, may be 0 for bootstrap server
    time_t                  lifetime;     // lifetime of the registration in sec or 0 if default value (86400 sec), also used as hold off time for bootstrap servers
    time_t                  registration; // date of the last registration in sec or end of client hold off time for bootstrap servers
    lwm2m_binding_t         binding;      // client connection mode with this server
    void *                  sessionH;
    lwm2m_status_t          status;
    char *                  location;
    bool                    dirty;
    lwm2m_block1_data_t *   block1Data;   // buffer to handle block1 data, should be replace by a list to support several block1 transfer by server.
} lwm2m_server_t;
```

`lwm2m_binding_t`和`lwm2m_status_t`分别记录了与该服务器的通信方式（例如`UDP`、`SMS`等）和与该服务器的通信状态（是否正在注册过程中等）。

`void * session`记录了与该服务器通信的会话信息，在示例程序中是`connection_t`类型。

### `lwm2m_observed_t`

#### 概述

记录客户端实体中被服务器观察（监听）的资源（事件）。

#### 结构体结构

```c
/*
 * LWM2M observed resources
 */
typedef struct _lwm2m_watcher_
{
    struct _lwm2m_watcher_ * next;

    bool active;
    bool update;
    lwm2m_server_t * server;
    lwm2m_attributes_t * parameters;
    uint8_t token[8];
    size_t tokenLen;
    time_t lastTime;
    uint32_t counter;
    uint16_t lastMid;
    union
    {
        int64_t asInteger;
        double  asFloat;
    } lastValue;
} lwm2m_watcher_t;

typedef struct _lwm2m_observed_
{
    struct _lwm2m_observed_ * next;

    lwm2m_uri_t uri;
    lwm2m_watcher_t * watcherList;
} lwm2m_observed_t;
```

`lwm2m_watcher_t`代表了观察者的相关信息；

`lwm2m_observed_t`中`uri`表示观察的资源URI，`watcherList`是观察该资源的观察者列表。

### `lwm2m_client_t`

#### 概述

该结构体描述了远端客户端的基本信息。

#### 结构体结构

```c
/*
 * LWM2M Clients
 *
 * Be careful not to mix lwm2m_client_object_t used to store list of objects of remote clients
 * and lwm2m_object_t describing objects exposed to remote servers.
 *
 */

typedef struct _lwm2m_client_object_
{
    struct _lwm2m_client_object_ * next; // matches lwm2m_list_t::next
    uint16_t                 id;         // matches lwm2m_list_t::id
    lwm2m_list_t *           instanceList;
} lwm2m_client_object_t;

typedef struct _lwm2m_observation_
{
    struct _lwm2m_observation_ * next;  // matches lwm2m_list_t::next
    uint16_t                     id;    // matches lwm2m_list_t::id
    struct _lwm2m_client_ * clientP;
    lwm2m_uri_t             uri;
    lwm2m_status_t          status;
    lwm2m_result_callback_t callback;
    void *                  userData;
} lwm2m_observation_t;

typedef struct _lwm2m_client_
{
    struct _lwm2m_client_ * next;       // matches lwm2m_list_t::next
    uint16_t                internalID; // matches lwm2m_list_t::id
    char *                  name;
    lwm2m_binding_t         binding;
    char *                  msisdn;
    char *                  altPath;
    bool                    supportJSON;
    uint32_t                lifetime;
    time_t                  endOfLife;
    void *                  sessionH;
    lwm2m_client_object_t * objectList;
    lwm2m_observation_t *   observationList;
} lwm2m_client_t;
```

*注意*将`lwm2m_observation_t`和`lwm2m_observed_t`这两个结构体区分开，前者用于在服务器端中记录对应客户端资源的观察情况，而后者则恰恰相反，用于在客户端中记录被服务器观察的资源。

`sessionH`成员是客户端和服务器的会话记录，在示例程序中是一个`connection_t`类型。

## LWM2M应用程序

### 概述

本节主要叙述`lwm2m`客户端、服务器程序的创建流程、工作方式和基本操作，具体的实现可以参照`wakaama`的`lightclient`和`server`示例程序。注意，`wakaama`本身并不参与具体的底层通信的实现，也就是说，你需要自己去实现`udp`通信或者其他通信。

### 需要使用的函数

#### 初始化

```c
// initialize a liblwm2m context.
lwm2m_context_t * lwm2m_init(void * userData);
// close a liblwm2m context.
void lwm2m_close(lwm2m_context_t * contextP);
```

#### 协议栈内部相关

```c
// perform any required pending operation and adjust timeoutP to the maximal time interval to wait in seconds.
int lwm2m_step(lwm2m_context_t * contextP, time_t * timeoutP);
// dispatch received data to liblwm2m
void lwm2m_handle_packet(lwm2m_context_t * contextP, uint8_t * buffer, int length, void * fromSessionH);
```

这一部分不包含内部通信

#### 客户端函数

```c
// configure the client side with the Endpoint Name, binding, MSISDN (can be nil), alternative path
// for objects (can be nil) and a list of objects.
// LWM2M Security Object (ID 0) must be present with either a bootstrap server or a LWM2M server and
// its matching LWM2M Server Object (ID 1) instance
int lwm2m_configure(lwm2m_context_t * contextP, const char * endpointName, const char * msisdn, const char * altPath, uint16_t numObject, lwm2m_object_t * objectList[]);
int lwm2m_add_object(lwm2m_context_t * contextP, lwm2m_object_t * objectP);
int lwm2m_remove_object(lwm2m_context_t * contextP, uint16_t id);

// send a registration update to the server specified by the server short identifier
// or all if the ID is 0.
// If withObjects is true, the registration update contains the object list.
int lwm2m_update_registration(lwm2m_context_t * contextP, uint16_t shortServerID, bool withObjects);

void lwm2m_resource_value_changed(lwm2m_context_t * contextP, lwm2m_uri_t * uriP);
```

#### 服务器函数

```c
// Clients registration/deregistration monitoring API.
// When a LWM2M client registers, the callback is called with status COAP_201_CREATED.
// When a LWM2M client deregisters, the callback is called with status COAP_202_DELETED.
// clientID is the internal ID of the LWM2M Client.
// The callback's parameters uri, data, dataLength are always NULL.
// The lwm2m_client_t is present in the lwm2m_context_t's clientList when the callback is called. On a deregistration, it deleted when the callback returns.
void lwm2m_set_monitoring_callback(lwm2m_context_t * contextP, lwm2m_result_callback_t callback, void * userData);

// Device Management APIs
int lwm2m_dm_read(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_result_callback_t callback, void * userData);
int lwm2m_dm_discover(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_result_callback_t callback, void * userData);
int lwm2m_dm_write(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_media_type_t format, uint8_t * buffer, int length, lwm2m_result_callback_t callback, void * userData);
int lwm2m_dm_write_attributes(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_attributes_t * attrP, lwm2m_result_callback_t callback, void * userData);
int lwm2m_dm_execute(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_media_type_t format, uint8_t * buffer, int length, lwm2m_result_callback_t callback, void * userData);
int lwm2m_dm_create(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_media_type_t format, uint8_t * buffer, int length, lwm2m_result_callback_t callback, void * userData);
int lwm2m_dm_delete(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_result_callback_t callback, void * userData);

// Information Reporting APIs
int lwm2m_observe(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_result_callback_t callback, void * userData);
int lwm2m_observe_cancel(lwm2m_context_t * contextP, uint16_t clientID, lwm2m_uri_t * uriP, lwm2m_result_callback_t callback, void * userData);

// result callback
typedef void (*lwm2m_result_callback_t) (uint16_t clientID, lwm2m_uri_t * uriP, int status, lwm2m_media_type_t format, uint8_t * data, int dataLength, void * userData);
```

### 如何创建客户端程序

首先我们需要调用`lwm2m_init`来创建客户端运行环境：

```c
lwm2m_context_t *client;
client = lwm2m_init(&userdata);
```

`userdata`可以是任意用户自定义的数据结构，它将被存储在`lwm2m_context_t`的`userdata`字段中。

然后调用`lwm2m_configure`来对客户端运行环境进行配置：

```c
char *name = "end point name"
lwm2m_object objArr[objCount] = {...};
result = lwm2m_configure(client, name, NULL, NULL, objCount, &objArr);
```

第一个参数是运行环境本身，第二个参数是终端节点名称，`objCount`指明了数组`objArr`的长度，`objArr`存储了该终端节点所含有的对象。

以上就完成了客户端节点的初始化工作，之后我们主需要编写客户端程序的主循环。

每次循环开始时，我们需要先让协议栈处理一些内部事务（比如发送或重发一些数据包）：

```c
struct timeval tv;
result = lwm2m_step(lwm2mH, &(tv.tv_sec));
```

`result`小于0时说明发生致命错误。

`lwm2m_step`的第二个参数是超时选项，默认是60秒。

然后通过套接字读取网络通信数据，并将数据使用`lwm2m_handle_packet`函数将数据交给协议栈进行处理即可：

```c
result = select(nfds, &rfds, NULL, NULL, &(tv));
if (result > 0) {
  nbytes = recvfrom(lfd, buffer, sizeof(buffer), (struct sockaddr *)&remote, &socklen);
  if (nbytes > 0) {
    lwm2m_handle_packet(client, buffer, nbytes, connP);
  }
}
```

`connP`是与该远端服务器通信用的会话实体，在`wakaama`的示例程序中，是`connection_t`类型。

以上即是创建客户端的流程，详细的实现细节可以参考`wakaama`的示例程序。

### 如何创建服务器程序

创建客户端和创建服务器程序的步骤类似，不过在初始化过程中略有区别：

```c
// 创建运行环境
lwm2m_context_t *server = lwm2m_init(NULL);

// 设置监控函数
lwm2m_set_monitoring_callback(server, monitoring_callback, client);
```

可以看到，我们不需要对服务器进行任何配置。只需要额外设置一个监控回调函数，该回调函数将会在任意客户端注册或注销时被触发。

服务器端主要的工作是通过设备控制和管理接口对客户端节点进行管理，每次通信的结果通过回调函数`lwm2m_result_callback_t`来接受，其原型如下：

```c
typedef void (*lwm2m_result_callback_t) (uint16_t clientID, lwm2m_uri_t * uriP, int status, lwm2m_media_type_t format, uint8_t * data, int dataLength, void * userData);
```

其中，`userdata`参数可以是任意自定义数据，通常我们将`lwm2m_context_t`类型，也就是服务器运行环境的引用传入。

我们可以通过各类设备管理和操作接口来与客户端进行通信：

```c
lwm2m_dm_read(server, 0, "/1/2/0", read_callback, server);
```

