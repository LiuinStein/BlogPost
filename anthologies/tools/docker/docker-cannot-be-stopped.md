### 0x00 问题简述

docker container根本停不下来，使用`docker rm -f`命令强行删除容器也不行，错误称我没有足够的权限，即便我是以`root`用户在运行docker。错误代码示例：

```
ERROR: for yattyadocker_web_1  cannot stop container: 1f04148910c5bac38983e6beb3f6da4c8be3f46ceeccdc8d7de0da9d2d76edd8: Cannot kill container 1f04148910c5bac38983e6beb3f6da4c8be3f46ceeccdc8d7de0da9d2d76edd8: rpc error: code = PermissionDenied desc = permission denied
```

### 0x01 起因

`Apparmor`由于某些未知原因无法正常工作引起，`Apparmor`是一个应用于Linux下的安全程序。

### 0x02 解决方案

```shell
systemctl disable apparmor.service --now
service apparmor teardown
```

### 0x03 参考资料

[Docker Containers can not be stopped or removed - permission denied Error](https://stackoverflow.com/a/51851856)