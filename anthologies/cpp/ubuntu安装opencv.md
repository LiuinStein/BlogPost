### 0x00 Ubuntu安装OpenCV for C++

首先安装OpenCV所需的库：

```sh
sudo apt-get install build-essential
sudo apt-get install cmake git libgtk2.0-dev pkg-config libavcodec-dev libavformat-dev libswscale-dev
sudo apt-get install python-dev python-numpy libtbb2 libtbb-dev libjpeg-dev libpng-dev libtiff-dev libjasper-dev libdc1394-22-dev
```

> 如果出现错误`unable to locate libjasper-dev`，则使用如下命令：
>
> ```sh
> sudo add-apt-repository "deb http://security.ubuntu.com/ubuntu xenial-security main"
> sudo apt update
> sudo apt install libjasper1 libjasper-dev
> ```
>
> 这个错误多发于Ubuntu18.04及以上系统中。

依赖安装完成之后，去官网http://opencv.org/releases.html下载最新版的OpenCV源码，然后解压，切换到解压后的目录中，然后执行如下命令：

```sh
mkdir build
cd build
cmake ../
make
sudo make install
```

编译安装的过程可能很慢，慢慢等待即可。

### 0x01 配置OpenCV for C++编译环境

安装完了就得配置，你说是吧，如果你直接在一个C++文件里面引入头文件：

```cpp
#include <opencv2/opencv.hpp>
```

首先，敲代码的时候，会没有OpenCV库相关的智能提示，我们可以在项目目录下的`.vscode`文件夹下创建文件`c_cpp_properties.json`，在里面输入如下内容：

```json
{
    "configurations": [
        {
            "name": "Linux",
            "includePath": [
                "${workspaceFolder}/**",
                "/usr/include/c++/7/**",
                "/usr/local/include/opencv4/**"
            ],
            "defines": [],
            "compilerPath": "/usr/bin/gcc",
            "intelliSenseMode": "clang-x64"
        }
    ],
    "version": 4
}
```

上述代码中真正起到作用的是那个`includePath`，它决定了Visual Studio Code的智能提示包含目录。

光整完了这些还不行，编译代码的时候，`g++`也有可能报错说找不到头文件，这个时候我们可以设置一个环境变量：

```sh
export CPLUS_INCLUDE_PATH=/usr/local/include/opencv4:$CPLUS_INCLUDE_PATH
```

这么设置相对于当前用户来说是临时的，重启系统之后可能就需要重新设置，对于`zsh`来说，我们可以在`~/.zshrc`文件的结尾添加上如上命令，即可永久地设置。

在设置这个的时候要注意一个坑，就是对于系统中不同的用户是有不同的`~/.zshrc`文件，我一开始给`root`用户设置了一下，然后在vsc中编译死活不成功，还是找不到头文件，结果后来才想到vsc是运行在普通用户下的，编译时的`g++`也是运行在普通用户下的，所以依旧找不到头文件。建议设置的时候，一定要注意`g++`是跑在什么用户下，再针对这个用户的`~/.zshrc`文件进行设置。

