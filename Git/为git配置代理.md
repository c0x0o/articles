# 为git配置代理

## git的三种配置形式

git拥有三种配置文件形式——local，global和system。

local配置适用于当前仓库，路径位于`./.git/gitconfig`，通过`git config --local`指定

global配置适用于当前登录用户，路径位于`~/.gitconfig`，通过`git config --global`指定

system配置使用于当前系统，路径位于`/etc/gitconfig，通过git config --system指定`

上述三种配置优先级从上至下依次减小，即local覆盖global和system，global覆盖system



## git的配置选项形式

所有的git配置项类似于：`foo.bar`，

例如：

1. `user.name` 你的用户名
2. `user.email` 你的邮箱
3. `core.editor` 你使用的编辑器



## 为git配置proxy

```shell
git config --global http.proxy http://proxyserver:1080
git config --global https.proxy https://proxyserver:1080
```

使用上述的命令就可以避免在全局环境下设置代理，如果使用`--local`选项更是可以为不同的仓库设置不同的代理，大大方便了墙内用户。