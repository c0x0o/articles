# 以正确的姿势在ubuntu中管理软件

## 概述

在ubuntu系统中，我们使用`apt(Advanced Package Tool)`作为包管理工具。通常情况下，想要安装一个软件，我们只需要知道包含该软件的软件包包名，然后使用`sudo apt install <package_name>`即可。但是当由于官方维护的apt源为了保证系统的稳定性，通常并不会激进地采用最新的版本的软件（虽然源里面提供最新版本的软件包，但并不会作为默认选项，而是独立于默认的软件包单独发布），而我们又希望能使用到最新的特性时，就需要自己安装对应的软件包（甚至自行编译并安装），然后进行一些设置。那么，如何以正确的步骤使得我们自行安装的软件与ubuntu更好的融合，以避免后期出现一些难以纠正的错误呢？请看后文我们详细的讨论。

## 目录结构

首先我们列举一些安装软件时重要的目录及其作用：

| 路径                   | 作用                                       |
| -------------------- | ---------------------------------------- |
| `/opt`               | 存放独立于系统的可选第三方软件，某些软件遵循此规则，如果你安装过google-chrome，那么你会发现，它会自动把自己安装在这个目录下 |
| `/bin`               | 存放系统级可执行文件的地方                            |
| `/usr`               | 存放自动安装或通过包管理器安装的软件包的目录                   |
| `/usr/bin`           | 存放可执行文件，但是很多情况下，开发者只是在其中创建一个软链接指向真正的程序入口。 |
| `/usr/share`         | 大部分软件都会把自己的本体目录安置于此目录下，然后通过软连接的方式将程序入口放置在`../bin`中 |
| `/usr/local`         | 类似与`C:\\program files`，其中的目录结构（以及对应作用）和`/usr`类似，用于存放手动安装的程序 |
| `/usr/include`       | 某些软件包不提供可执行文件，而是作为代码库存在，那么其头文件应当被存放于此处，便于被系统搜索 |
| `/usr/local/include` | 同上                                       |
| `/usr/share/man`     | `man`命令本身也是一个程序，其目录中的`man1`等目录代表了`man`命令的不同区，如果你想要安装的软件包包含了man page，那么你应该在对应的`man`目录下为其创建一个软连接 |

## 软件的安装

### 从官方源安装

如果你很幸运的发现，你需要的软件包正好被官方源所维护，那么你只需要执行：

```shell
sudo apt install <package_name>
```

即可。

顺便一提，ubuntu软件包通常遵循以下命名规则：`<package_name>-<version>-<dev>`。`<version>`是指“3.9.0”这样的字符串，`<dev>`标识是否是开发版软件包，其中包含了基于此软件包的头文件和库文件。比如`python-3.3`是指3.3版的python，而`python-3.3-dev`则包含了`python.h`头文件和库文件。

### 从个人源安装

Personal Package Archives（个人软件包档案）是Ubuntu Launchpad网站提供的一项服务，允许个人用户上传软件源代码，通过Launchpad进行编译并发布为2进制软件包。如果你有一个类似`ppa:username/ppa-name`这样的链接，比如`ppa:jonathonf/vim`（这是vim的开发者的个人源，你可以从这个源安装最新vim），你可以：

```shell
sudo add-apt-repository ppa:jonathonf/vim
sudo apt update
sudo apt install vim
```

### 从deb文件安装

如果你拥有（或者可以获取）软件的针对apt的打包版本——`deb`文件，那么你可以通过：

```shell
sudo dpkg -i <deb_file>
```

命令来安装此软件包，这种安装方式与从官方源安装没有太大区别。

### 从源码包安装

但如果官方源中没有你想要的软件包，而且软件开发者也没有提供安装包，那么你需要从源码包来安装软件。请遵循以下步骤：

1. 编译你的软件包，具体说明请遵循软件开发者的指引，通常完成编译的软件会在`bin`和`lib`目录下生成你需要的软件或库文件
2. 接着，将整个软件目录拷贝（或链接）至`/usr/local/share`下
3. 如果是可执行程序，请将编译好的可执行文件（通常在`bin`目录下）链接至`/usr/local/bin`下
4. 如果是库文件，请将库文件放置在`/usr/local/lib`下，头文件放置在`/usr/local/include`下
5. 拥有man page的软件，可以将man page链接到`/usr/local/share/man`对应的区下，若对应区目录不存在，则自行创建

注意！！！按照上述规则将软件文件夹放置在`/usr`目录下也是可行的。

## 系统默认软件和多版本共存

### 系统默认软件

类似于Windows中默认软件（比如默认浏览器、默认编辑器等）；ubuntu也定义了若干默认程序，比如：默认编辑器命令`editor`，默认c编译器命令`cc`，默认c++编译器命令`c++`，默认浏览器命令`x-www-browser`，这些命令实际上是定义在`/usr/bin`中一系列链接，通过修改这些链接的指向来修改默认程序的目的。

为了方便的进行这种指定操作，ubuntu系统提供了一个`update-alternatives`命令。这个命令不仅可以修改已经有的默认命令，还可以根据用户需要创建新的默认命令。

#### 创建新的默认命令

```shell
update-alternatives --install link name path priority [--slave link name path]
```

无论是创建新的默认命令，还是为已有的默认命令更新新的可选项，都需要使用此命令。

`link`指代默认命令在系统中的位置，例如`/usr/bin/editor`；`name`指代符号链接的名字，这个符号链接存储在`/etc/alternatives`中，所有的默认链接会先链接到此处，再由此处的符号链接链接到真实的可执行文件处；`path`指代真实可执行文件的位置；`priority`是指创建的可选项的优先级，系统在默认情况下下，会选择优先级最高的那一项。

对于可选的“从连接”，必须跟现有的默认命令的从链接个数一致，否则在添加时会导致错误（通过添加`--force`来忽略这个错误）。一个从链接的应用例子是，当更换默认编译器时，我们可能不仅需要把`cc`的例子从`gcc`变为`clang`，还需要把`cc`的man page修改为`clang`的man page。`--slave`选项可以指定多次。

因此，我们在安装完程序后，如果将来涉及到默认程序或者多版本共存时，请使用`update-alternatives`。

#### 更换默认程序

```shell
update-alternatives --set name path
update-alternatives --config name
```

`--set`选项用于直接切换一个默认程序的指向（`path`必须已经被添加），例如`update-alternatives --set cc /usr/bin/gcc`；`--config`选项则是提供一个交互式的方式来从多个可选项中选择一个作为默认程序。通过手工切换的程序将会忽略优先级设置。

#### 移除默认程序

```shell
update-alternatives --remove name path
update-alternatives --remove-all name
```

`--remove`用于删除默认程序的一个选项；`--remove-all`则是直接删除这一默认命令及其全部选项。

#### 杂项

```shell
update-alternatives --all
update-alternatives --auto name
update-alternatives --display name
```

`--all`选项相当于对每一个默认命令调用`--config`；`--auto`选项用于将一个默认命令重置为自动模式，即忽略手工的选择，转为以优先级为选择标准；`--display`选项用于显示一个默认命令的主从链接的信息。

### 多版本共存

有了系统默认软件这一工具，我们可以直接将不同版本的命令作为一个默认软件的备选项进行添加，例如：

```shell
update-alternatives --install /usr/bin/cc cc /usr/bin/gcc-4 1024 \
--slave /usr/share/man/man1/cc.1.gz cc.1.gz /usr/share/gcc-4/man/man1/gcc.1.gz

update-alternatives --install /usr/bin/cc cc /usr/bin/gcc-5 1024 \
--slave /usr/share/man/man1/cc.1.gz cc.1.gz /usr/share/gcc-5/man/man1/gcc.1.gz
```

添加完成后，就可以通过`update-alternatives --auto`或者`update-alternatives --set`来进行版本切换了。

## 升级软件

### 通过源进行升级

非常简单，just:

```shell
sudo apt update
sudo apt upgrade
```

上述命令会更新系统中全部的包，如果你不想更新某些包：

```shell
sudo apt-mark hold <package_name>

# 取消上述设定
sudo apt-mark unhold <package_name>
```

### 手动升级

更新源代码然后重新编译安装吧，少年

### 源码包升级

## 移除软件

### 通过源和deb包安装的软件

直接执行：

```shell
sudo apt remove <package_name>
```

但是该命令不会删除它遗留在系统中的配置文件，如果我们还想移除配置文件，则：

```shell
sudo apt remove --purge <package_name>
```

然后，我们还需要经常清除系统中不再需要的依赖：

```shell
sudo apt autoremove
```

如果实在需要更多的硬盘空间，你可以删除`apt`下载包时所产生的缓存：

```shell
sudo apt autoclean
```

### 通过源码安装的软件

这个就需要你自己去手工处理一切，包括：

1. 删除可执行文件的软连接
2. 删除可能的头文件和库
3. 利用`update-alternatives`删除默认软件可选项
4. 删除软件本体文件夹

当然，如果软件开发者提供了自动的安装和卸载的脚本那就是最吼的啦。