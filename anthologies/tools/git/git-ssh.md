### 0x00 Git配置`ssh`登陆

记录一下Git配置`ssh`登陆的方式，以便备忘。

首先我们需要使用`ssh-keygen`来生成一对公私钥：

```sh
ssh-keygen -t rsa -C shaoqunliu@sogou-inc.com -b 4096
```

输入命令之后，程序会提示你输入密钥密码和密钥存放地址信息。然后你就可以在你所设置的目录下找到一个没有扩展名的私钥文件（默认名称为`id_rsa`），以及一个扩展名为`pub`的公钥文件。

此后我们需要使用`ssh-add`命令将生成的私钥文件添加到`ssh`：

```
ssh-add H:/workspace/_keys/id_rsa
```

如上所示，后面的路径即为私钥文件的地址。

> TL;DR
>
> 如果你在执行上述命令的时候，遇到如下错误信息：
>
> ```
> Could not open a connection to your authentication agent.
> ```
>
> 可以试着执行一下下面这个命令，成功后错误即可消除：
>
> ```sh
> eval ssh-agent bash
> ```

此后，我们再将公钥信息通过网页端添加至GitHub或者GitLab，添加完成后，我们就可以使用`git clone`命令来clone一个我们私有的项目来检验相关配置是否正确完成。