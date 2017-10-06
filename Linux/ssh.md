# SSH

## 简介

在Linux系统中，OpenSSH是目前最流行的远程系统登录与文件传输应用，也是传统Telenet、FTP和R系列等网络应用的换代产品。与Telnet、FTP等使用明文传输的阮家不同，SSH采用非对称加密技术来加密通信双方的数据，在正式开始传输数据之前，双方首先要交换密钥，当收到对方的数据时，再利用密钥和相应的程序对数据进行解密。

## 涉及的软件

1. ssh：用于登录远程主机
2. ssh-agent：帮助你管理机器上的私钥和公钥
3. ssh-keygen：生成密钥对
4. ssh-keyscan：搜集若干主机上的公钥
5. ssh-copy-id：登录远程主机并把自己的公钥添加到authorized\_keys文件中
6. scp：在两个主机之间传递文件，远程安全的`cp`命令

## 涉及的目录和文件

1. ~/.ssh/id\_xxx
2. ~/.ssh/id\_xxx.pub
3. ~/.ssh/authorized\_keys
4. ~/.ssh/known\_hosts
5. /etc/hosts.equiv
6. /etc/shosts.equiv
7. ~/rhosts
8. ~/shosts

## 从ssh命令开始

> 再继续接下来的阅读之前，你需要对非对称加密和网络通信有最基本的了解，如果你对这方面的知识不太熟悉，建议你先去学习相关的知识点

我们可以使用ssh命令来登陆一个主机：
```shell
ssh username@hostname -p port
```

也可以只是执行一条命令：
```shell
ssh username@hostname command
```

当不指定`username`时，使用当前shell的用户名作为登录用户名。ssh协议目前有两个版本，第一个版本已经不建议使用（因为该协议具有密码学上的漏洞）。

ssh支持如下几种登录验证方式：
1. password authentication：也就是我们常用的密码登录
2. GSSAPI-based authentication：GSSAPI是一组安全事务的应用程序接口，因实现不同而采用不同的加密方式
3. host-based authentication：通过`hosts.equiv`，`shosts.equiv`，`rhosts`，`shosts`文件中规定的主机将可以直接登录本机，但是会使用`known_hosts`文件中的主机指纹来认证请求发起方的身份
4. public key authentication：公钥认证，通过在远程主机预先添加的本机公钥（`~/authorized_keys`），来认证本机的身份从而登录
5. certificate authentication：证书登录，也是一种基于公钥/私钥的认证方式。最常见的证书认证是HTTPS协议，这是一种客户端认证服务器的应用方式；ssh这里用证书进行服务器对客户端的认证过程。
6. challenge-response authentication：挑战应答认证。实际上是密码认证的超集（密码认证就是一种挑战应答认证），不过它会使用一些诸如PAM等后台机制来完成这一过程，由于了解不多，这里不展开叙述。

鉴于笔者水平，在本文中只会展开叙述密码认证、certificate authentication和public key authentication这三种认证方式。

## 密码认证

这大概是世界上使用最广泛，也是最不安全的认证方式之一——因为密码认证的安全性，很大程度上取决于密码的复杂性。

通用的安全密码应当至少符合一下几个要求：

1. 至少包含8个字符
2. 混合了数字、大写字母、小写字母、特殊字符
3. 不能含有明显且为大多数人所知的规律（也就是我们常说的弱口令），比如`12345678`，`p@ssw0rd`和一些常用硬件设备的默认出厂密码，这些密码往往看起来比较复杂，但是由于使用范围过广（为大多数人/黑客所知），因此在基于字典的暴力攻击下不具有任何安全性

但是由于复杂的密码不能被人轻易地记住，为了满足上述要求，人们通常会基于自身的信息来设置一些复杂密码（比如生日，重要短语的缩写，姓名的缩写等），这又导致黑客有可能在各种渠道获取用户的个人信息加以猜测，从而增加暴力破解的成功率。所以密码认证的安全性往往难以得到保障。

最佳的实践是使用密码管理软件来管理你的密码，它能随机生成复杂度极高的密码，而你只要每次从中读取你需要的密码即可。但是如何保障此软件的安全，又成了一个新的难题...所以作为一个优秀的Hacker，应当尽量避免使用密码认证登录远程主机，而是使用后面叙述的两种认证方式。

## public key authentication

所谓公钥认证，即是远程主机通过预先配置的客户机公钥，对客户端进行挑战（一般是用客户机公钥加密一个随机数），如果客户机能成功相应该挑战（即可以成功解密该数据包获取指定的随机数），那么即可以验证客户机的身份，整个过程都是被加密的。

具体的实施步骤是：
1. 客户机生成自己的密钥对
2. 讲公钥添加目标远程主机的`authorized_keys`文件中
3. 修改远程主机的配置文件并重启sshd服务

### 生成密钥对

我们可以使用`ssh-keygen`命令来生成密钥对，ssh支持多种类型、不同长度的非对称加密技术，例如：RSA，DSA，ED25519。

```shell
# 生成一对4096Bit长度的RSA秘钥对
ssh-keygen -t rsa -b 4096 -C "your_email.com"
```

`-t`参数用于指定加密算法，`-b`参数用于指定生成的密钥长度，`-C`参数指定跟这个密钥相关的注释信息，通常是你的联系方式（如邮箱），当然还有一些额外的参数，你可以在它的交互式程序中完成其余的步骤。

如果你在交互式程序中使用了默认设置，那么新生成的密钥对会生成在`~/.ssh`路径下，生成的文件名满足`id_<algorithm>[.pub]`。比如，对于之前的命令，将会生成私钥文件`id_rsa`和公钥文件`id_rsa.pub`。

现在，你已经拥有了足以证明你自己（或者你的一台主机）身份的密钥对。

TIPS：交互式程序中需要你输入激活这个密钥的口令，如果你在后面想要更改这个口令，可以使用命令`ssh-keygen -f ~/.ssh/id_rsa -p`。

### 配置远程主机

成功生成了你自己的密钥之后，我们需要让远程主机能够通过公钥来识别我们的身份。为了完成这一步，我们需要把我们的公钥（注意，不是私钥，私钥应当只能被你一人所访问）添加到远程主机对应账户的`authorized_keys`文件中。当然，在此之前，你应该拥有写`authorized_keys`文件的权限（无论这种权限是合法的，还是非法的）。

第一种比较原始的方法是，我们手动把公钥内容*附加*到`authorized_keys`文件当中，注意不要覆盖该文件，否则将会导致所有的已有授权信息丢失。

```shell
cat your_pub_key >> ~/.ssh/authorized_keys
```

你也可以在图形界面进行复制粘贴，但是需要注意的是，不要忽略的结尾的换行，否则将会导致验证失败。

第二种方法是通过`ssh-copy-id`命令来自动完成这一步骤。

```shell
ssh-copy-id -i keyfile -p port username@host
```

这一命令胜在简单有效，但是你必须已经拥有了对应账户的shell权限。另外`keyfile`不需要携带`.pub`后缀，也可以不指定该参数，默认使用`id_xxx`。

成功添加了公钥之后，我们还需要一点点设置工作——打开密钥认证开关。

打开`/etc/ssh/sshd_config`文件，找到或添加以下行：
```
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys .ssh/authorized_keys2
```

如果希望在启用公钥验证的同时关闭密码验证，找到或添加以下行：
```
PasswordAuthentication no
PermitEmptyPasswords no
ChallengeResponseAuthentication no
```

最后，重启sshd服务：`service sshd restart`

## certificate authentication

提到证书，就不得不先讲述一下CA（certifying authority，认证机构）这个概念。

通过之前的讨论我们知道，公钥是认证某一链接对端身份的重要手段——如果对方发送的公钥符合本机中所存储的公钥，那么我们就可以认为对方的身份是可信的。但是考虑这样一种情况，如果从第一次链接伊始，我们和服务器的链接就被某恶意的第三方劫持，服务器和我们获取的其实都是攻击者的公钥，但却被误以为是可信的身份，这就造成了中间人攻击。此时，中间人可以伪装成服务器和请求者任意一端向另一端发起“可信的”请求。为了有效避免中间人攻击，人们提出建立许多被大家认可的权威机构（CA）。当我们需要认证（下称client）某一个对端（下称server）的身份时，需要向该权威机构请求server的公钥（而不是直接向它本人请求）。权威机构使用自己的私钥加密该server的公钥（这一过程称为签名），而client则使用权威机构的公钥来解密该消息病获得server的可信公钥，这个过程可以很大程度上避免中间人攻击。

而为了避免“蛋生鸡，鸡生蛋”悖论，通常情况下，权威机构的公钥是被预装在操作系统或软件之中的（比如你可以在FireFox浏览器的安全设置中找到它所信任的CA证书）。国内的证书机构有CFCA（金融安全认证中心），CTCA（中国电信认证中心），海关认证中心（SCCA）等。

如果有条件，你可以向CA申请托管服务器的公钥（特别是你的服务器对公众提供内容）。如果不能，或者在局域网内部自建CA服务器时，则需要自己对自己的公钥进行签名。注意，正常情况下你不应该相信服务器的自签名证书（这种证书没有在公共CA上注册），除非你知道这种情况的风险。

在ssh中，certificate authentication的实质是使用一对签名密钥，对用户公钥进行签名，然后把签名密钥添加到服务器的信任列表之中。这样当用户发起登录请求时，服务器使用信任列表中的私钥来验证用户证书，验证成功则可以登录。

如上所述，我们需要至少2对密钥来完成上述流程：一对存储在服务器端或者局域网内部的CA服务器，用于为用户公钥签名；另一对则是用户密钥。

假设你先在已经拥有了签名密钥`ca_key`和用户密钥`id_rsa`。首先我们在本地使用签名密钥对用户密钥进行签名：

```shell
ssh-keygen -s ca_key -I user_yourname -n yourname -V +52w id_rsa.pub
```

`-s`参数指明签名密钥的私钥；`-I`参数指明证书的名字（标识符）；`-n`指明该证书认证的用户名；`-V`参数指明该证书的有效期，这里的+52w说明该证书有效期是52weeks，即一年；`id_ras.pub`是待签名的公钥。

完成用户公钥的签名之后，我们把签名用密钥对拷贝至服务器的`/etc/ssh`目录下，然后在`sshd_config`中添加（找到并修改）如下行：

```
TrustedUserCAKeys /etc/ssh/ca_key.pub
```

最后，最好删除掉本地的签名密钥对或者找一个安全的地方存储它们以备后续使用。

## ssh相关

### scp

`scp`命令可以在远程主机之间拷贝命令，并且使用加密的信道传输数据。它的使用方式和基本参数与`cp`命令没有区别，你可以像使用`cp`命令一样使用它，唯一的却别是在进行拷贝时需要指明主机名且需要进行身份认证：

```shell
# 拷贝remoteserver中remoteUser的.bashrc文件到当前目录下
scp remoteUser@remoteserver.com:/home/remoteUser/.bashrc .

# 拷贝本地的.vim文件夹到远程服务器上
scp -r ~/.vim remoteuser@remoteserver.com:/home/remoteuser/
```

### vim编辑远程文件

这个是非常有用的一个小技巧，如果你只是像编辑远程服务器上的一个文件，你可以使用这个命令来减少你的操作：

```shell
vim scp://remoteuser@remoteserver//home/remoteuser/.bashrc
```

### 为用户端添加服务器认证

#### 通过公钥认证

将服务器公钥直接添加到`~/.ssh/known_hosts`文件中即可。注意，一个公钥一行。下次登陆之前，如果服务器公钥不符，则无法登录该主机；如果你明白其中的风险，可以从`known_hosts`文件中删除对应的行。也可以使用`ssh-keygen -R host`命令来删除。

TIPS：你可以使用`ssh-keygen -r host`来查看属于某一主机的公钥

#### 通过证书认证

首先你需要为服务器生成一个证书，生成host证书和生成用户证书有所不同，需要在命令中加入`-h`选项：`ssh-keygen -s ca_host -I host_hostname -h -n hostname host_pubkey.pub`。然后，在服务器端的`sshd_config`文件中添加（找到或修改）：
```
HostCertificate /path/to/host_pubkey-cert.pub
```

最后，重启服务器端的sshd服务。此时服务器端已经在使用我们指定的证书了。

接下来，为了在客户端能够成功验证这个证书，我们需要把托管服务器公钥的CA公钥加入到本地`known_hosts`文件中，格式和一般的公钥略有区别：
```
@cert-authority *.remotehost.com ssh-rsa pubkey_string user@remotehost
```

现在，当你登录该主机时，将不会再提示是否信任该主机，因为此时ssh已经通过CA的公钥认证了该服务器的合法身份。
