# 基于显卡的Hash破解工具——Hashcat

## Hash函数概述

通常情况下，我们不会在系统中以明文存储密码，而是通过Hash函数来处理明文密码，然后在存储介质上存放Hash值用于口令登录。每次登陆的的时候，我们使用相同的算法（和salt）来计算用户输入口令的hash值，并与介质中存储的hash值进行比对（也就是说比对的不是口令明文，而是口令的hash值）。这样，攻击者仅能从系统中获取到用户口令Hash后密文，这种密文并不能直接用于系统的登录，同时由于Hash函数本身是单向函数，在获取密文的情况下，几乎不可能从密文恢复出明文，从而保证了口令本身的安全。

但是由于Hash函数的特性——“针对不同长度的输入，总是给出固定长度的输出”，这相当于将无限的输入，映射到了可以枚举的有限空间之中，那么我们从理论上必然可以找到两个不同的输入，经过Hash之后会得到相同的Hash值，这种情况被称为“碰撞”（collision）。这种情况对于Hash函数是致命的，因为你肯定不想别人输入了一个和你的口令毫不相关的字符串从而登录到你的银行账户。虽然Hash函数本身被设计为使这种碰撞出现的概率几乎为0，但是从目前来看，常用的MD4和MD5算法都被从算法层面上找到了寻求碰撞的方法（MD4在1991年就被发现有重大缺陷，MD5则是近几年被中国的龙小云教授攻破）。不过，找到Hash函数算法方面的突破口仍然是极为困难的。但是，当口令本身的强度不足（采用字符集太小，口令本身是常用的弱口令）时，我们能在可以接受的时间代价内，采用暴力的方法（brute-force）枚举明文空间，和获取的密文进行比对，从而恢复出对应Hash值的明文。说的直白一点，就是不断尝试各种可能的明文，看能不能得到相同的Hash值。

## Hashcat概述

实现上述攻击的工具数不胜数，但是很少有Hashcat这样功能完善、定制性强、性能极佳的工具。通过OpenCL提供的兼容能力，可以充分使用硬件的CPU、GPU、协处理器等一切当前可用的计算能力来帮助我们加快暴力破解的过程。你可以通过他们的[github地址](https://github.com/hashcat/hashcat)来获取源代码，当然你还需要在你的系统上安装OpenCL环境，Ubuntu16.04的用户可以参考[这篇文章](https://askubuntu.com/questions/850281/opencl-on-ubuntu-16-04-intel-sandy-bridge-cpu)。

Hashcat还支持非常多的攻击方式，比如字典攻击，暴力破解，组合攻击（多个字典共同作用），混合攻击（基于字典的暴力破解）等。

## 使用方法

> 多用用：
> hashcat --help

### 基础使用方法

我们假设你获取到的是最常见的32位普通MD5 Hash值（即使是MD5，也有非常多的版本和标准），并采用直接字典破解：

```shell
hashcat -a 3 -m 0 your_hash your_dict.dict
```

`you_hash`也可以是一个文件名，该文件每一行包含一个hash

所谓的字典文件，实际上是由若干行字符串组成的文件，每一行代表了一个可能的明文。直接字典破解就是依次尝试每个可能的明文。hashcat处理200M的字典文件大概只需要几秒。

### 常用参数

#### 攻击模式

`-a --attack-mode`指明了hashcat采用的攻击模式，可用的攻击模式如下：

|val|mode|description|
|---|----|-----------|
|0|straight|直接字典攻击，遍历字典的每一行|
|1|combination|组合攻击，多个字典共同作用，其中的值相互组合|
|3|brute-force|暴力破解，根据定制的Mask进行暴力破解|
|6|Hybrid wordlist + Mask|混合攻击，以字典文件为基础，通过Mask规则生成新的字典来进行攻击|
|7|Hybrid Mask + wordlist|混合攻击，以Mask规则为基础，通过append字典文件来进行攻击，与上一种刚好相反|

Example：

```shell
hashcat -a 0 -m 0 hashcode dict
hashcat -a 1 -m 0 hashcode dict1 dict2

# ?a?a?a?a?a?a是Mask规则，代表6位的可打印字符，在后文详细叙述
hashcat -a 3 -m 0 hashcode ?a?a?a?a?a?a

# 对于Hybrid攻击，我们首先要生成字典，然后再用该字典进行攻击
# 生成字典
hashcat -a 6 dict ?d?d?d > dict1 #在字典的每一行后添加3个0-9整数的枚举，生成新的字典
hashcat -a 7 ?d?d?d dict > dict2 #与上相反

# 使用新字典进行攻击
hashcat -a 0 -m 0 hashcode dict1
```

Hybrid攻击实际上是通过规则来生成新字典，比如`passwd`和`?d?d?d`组合会生成`passwd001`等新的行

#### Hash类型

hashcat支持一百多种hash类型的破解，通过`-m --hash-type`来指定，默认值为0，即md5。更多的选项请参考`hashcat --help`。如果你不知道你的hash值是哪一种类型，可以参考[这个网址](https://hashcat.net/wiki/doku.php?id=example\_hashes)

#### Mask规则

Mask规则类似于正则表达式，每一个元字符以`?`开头，代表明文中的一位；每一个元字符是一个字符集。以下是元字符列表：

|元字符|字符集|
|----|-----------|
|l|abcdefghijklmnopqrstuvwxyz|
|u|ABCDEFGHIJKLMNOPQRSTUVWXYZ|
|d|0123456789|
|h|0123456789abcdef|
|H|0123456789ABCDEF|
|s|` !"#$%&'()*+,-./:;<=>?@[\]^_\`{|}~`|
|a|?l?u?d?s|
|b|0x00-0xff|

如果我们的明文是6位整数，可以使用`?d?d?d?d?d?d`来暴力破解；如果我们是6位可打印字符，则`?a?a?a?a?a?a`。但如果是小写字母和数字的集合该怎么办呢？默认元字符中没有这个选项。我们可以使用自定义字符集来解决这个问题，hashcat支持在一个任务中最多4个自定义字符集，可以在命令行中用`-1 -2 -3-4 选项来定义`，并用`?1 ?2 ?3 ?4`来进行引用。比如，`hashcat -a 3 -m 0 -1 ?l?d md5code ?1?1?1?1?1`可以用来暴力破解明文为小写字母和数字的5位md5 Hash值。

当然你也可以在Mask规则中加入常规字符，比如`passwd?d?d?d`代表了`passwd001`到`passwd999`的字符串范围。

Mask规则实际上间接规定了明文的长度，但是更常见的情况是，我们指定了6位整数的搜索空间`?d?d?d?d?d?d`（hashcat只会搜索6位的明文，），明文却是5位整数。为了能让hashcat自动帮我们搜索更短的空间，而不是运行6次不同长度Mask的命令，我们可以指定`--increment`选项，比如`hashcat -m 0 -a 3 --increment md5hash ?d?d?d?d?d?d`将会搜索从1-6位所有的明文空间，相当于运行了：

```shell
hashcat -m 0 -a 3 md5hash ?d
hashcat -m 0 -a 3 md5hash ?d?d
hashcat -m 0 -a 3 md5hash ?d?d?d
hashcat -m 0 -a 3 md5hash ?d?d?d?d
hashcat -m 0 -a 3 md5hash ?d?d?d?d?d
hashcat -m 0 -a 3 md5hash ?d?d?d?d?d?d
```
