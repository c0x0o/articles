# C语言中的Volatile关键字

## 声明`volatile`关键字

```C
volatile int a;
// equal
int volatile a;
```

## 作用

`volatile`关键字声明该变量是易变的。所谓“易变的”变量是指在当前代码段没有显示的
改变该变量时，该变量的值会自动发生变化。这种情况可以发生在以下几个场景里：
1. 该变量对应的内存是一段经过映射的内存
2. 该变量可能被并发线程访问并修改
3. 改变量可能会被中断处理程序修改

比如下面的一段代码，我们假设变量`a`会被并发访问并修改：

```c
int *a = (int *)0xdeadbeaf

do {
  // do something
} until (0 == *a);
```

在我们的认知中，该代码将会不停的循环，直到`*a`被外部线程修改为0。同时，汇编后的
代码如下所示（仅作展示）：

```asm
  # 赋值
  movl $0xdeadbeaf, %eax
  movl $eax, (%edx)

loop:
  # do something
  movl (%edx), %eax
  jnz (%eax)
```

通常情况下，上面的代码将会很好的执行我们的意图（暂时忽略锁的问题），但是当我们
在一些实现不够好的编译器上开启优化选项后，该代码将会被优化为如下：

```asm
  # 赋值
  movl $0xdeadbeaf, %eax
  movl $eax, (%edx)
  movl (%edx), %eax

loop:
  # do something
  jnz (%eax)
```

我们可以看到，取`0xdeadbeaf`地址的值的操作只在循环开始时执行了一次，这将使得即
使`a`指向的值被修改，循环也永远不会退出。这是因为编译器做优化时，无法知道外部
线程会修改该值（临近的或者相关的代码中没有修改这一内存的操作），因此它认为这个
值是一个永远不会变的值，所以在优化操作中仅读取了一次值——而这显然不是我们需要的
行为。我们可以通过使用`volatile`关键字修饰`a`变量，使其成为指向一个`易变的int
内存`的指针，从而避免错误的优化：

```c
volatile int *a = (int *)0xdeadbeaf

do {
  // do something
} until (0 == *a);
```
