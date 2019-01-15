### 0x00 前言

使用SSH密钥登录服务器一是可以免密码登录，二是相对密码来说更加安全一些，只要你可以保证你的私钥文件不被泄露即可

### 0x01 生成SSH密钥

#### Linux

在Linux下，我们可以使用`ssh-keygen`来生成一对密钥，一般使用的参数只有两个

```shell
-b	指定密钥的长度，我一般使用2048位的密钥，即-b 2048
-t	指定加密算法，一般使用RSA算法，即-t rsa
```

在生成密钥完成之后会提示你输入一个密码，这个密码用来加密你的私钥，密码是可选的，但是，为了安全性着想，我们最好还是为其设置一个密码。

生成完成之后，会输出公钥和私钥的存储路径

#### Windows

在Windows下，我们可以使用puttygen来生成密钥，puttygen是带有图形化用户界面的，使用起来也相对简单，我们点击Generate按钮之后，然后用鼠标在puttygen窗体里面随便滑动（好像是为为了产生更随机的随机数），就可以生成一对密钥，如图

![](https://bucket.shaoqunliu.cn/image/0123.png)

### 0x02 将公钥上传至Linux服务器

#### Linux

可以通过scp命令将你的公钥上传至服务器，例如：

```
scp 公钥文件路径 用户名@主机名或IP地址:
```

#### Windows

Windows下可以使用WinSCP或Xftp等工具，直接将公钥上传至服务器

### 0x03 配置公钥登录

设置SSH，打开密钥登录功能

```shell
vi	/etc/ssh/sshd_config
# 打开密钥登录功能
RSAAuthentication yes
PubkeyAuthentication yes
# 设置root账户是否可以使用密钥登录
PermitRootLogin yes
# 禁用密码登录
PasswordAuthentication no
# 重启SSH服务
service sshd restart
```

配置公钥：

```shell
cd ~
mkdir ~/.ssh	# 可以通过ll -a来查看一下是否有这个文件夹，如果有就不用创建了
chmod 700 ~/.ssh	# rwx --- ---
mv 公钥文件路径 ~/.ssh	# 将公钥文件移动至~/.ssh
cd	~/.ssh
cat 公钥文件.pub >> authorized_keys
cat 公钥文件.pub >> authorized_keys2
chmod 600 authorized_keys2	# rw- --- ---
```

这样就完成了公钥登录的配置，然后我们就可以使用ssh命令（Linux）或XShell（Windows）来进行登录了。