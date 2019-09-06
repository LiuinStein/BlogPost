### 0x00 配置文件撰写规则 

首先给出其官方文档地址：

[Configuring a registry](https://docs.docker.com/registry/configuration/)

[Compose file version 3 reference](https://docs.docker.com/compose/compose-file/)

我们先来明白registry配置属性的覆盖规则，有两种：

第一种，如果你不想复写整个registry配置文件，而是想仅仅改变其中的几个属性。可以将[Configuring a registry](https://docs.docker.com/registry/configuration/)中给出的`yml`文件配置属性转换为相应的环境变量，然后在`docker run -e`命令中或者在`docker-compose.yml`中的`environment`标签下进行复写即可，基本规则为，首先对应的环境变量的名称以`REGISTRY`开头，然后按照`yml`文件中的层级，从上往下将名称转换为全大写，然后依次列出，中间以`_`作为分隔，例如如下`yml`配置的环境变量转换：

```yml
storage:
  filesystem:
    rootdirectory: /somewhere
```

转换的环境变量即为：

```
REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/somewhere
```

第二种，复写整个registry配置文件，首先依据其官网给出的[示例registry配置文件](https://github.com/docker/distribution/blob/master/cmd/registry/config-example.yml)和相关规则，创建一个你自己的registry配置文件，假设其名为`config.yml`，放在当前目录下，这样我们就可以通过container内外的文件映射方式来进行相关配置的复写，在`docker run`命令下的示例为：

```sh
docker run -d -p 5000:5000 --name registry \
-v `pwd`/config.yml:/etc/docker/registry/config.yml \
registry:2
```

注意其中的`-v`标签，同样我们还可以在`docker-compose.yml`文件中的`volumes`标签下进行配置，示例如下：

```yml
version: "3"
services:
  registry:
    image: registry:latest
    volumes: # 注意
      - `pwd`/config.yml:/etc/docker/registry/config.yml
```

### 0x01 为docker registry配置阿里云oss对象存储

我们的目的旨在实现将通过`docker push`命令上传到我们私有docker registry的image全部存储到阿里云的oss对象存储服务中。首先呢，我们登陆阿里云的官网，获得一份由阿里云提供的`ACCESSKEYID`和`ACCESSKEYSECRET`，以及你的对象存储的bucket名称及存储地域标签，docker registry项目内置了阿里云oss的存储驱动程序，所以我们直接配置即可！在此处我使用的是环境变量配置方法，`docker-compose.yml`文件中的配置如下：

```yml
environment:
  # Store in Aliyun OSS
  - REGISTRY_STORAGE=oss
  - REGISTRY_STORAGE_OSS_ACCESSKEYID= # access key id
  - REGISTRY_STORAGE_OSS_ACCESSKEYSECRET= # access key
  - REGISTRY_STORAGE_OSS_REGION= # oss的地域标签
  - REGISTRY_STORAGE_OSS_BUCKET= # 你的对象存储bucket名
```

给出可以参考的官方文档：

> [storage -  Configuring a registry](https://docs.docker.com/registry/configuration/#storage)
>
> [Aliyun OSS storage driver](https://docs.docker.com/registry/storage-drivers/oss/)

### 0x02 配置registry redis cache

此处的redis cache是为docker registry准备的，其官网对其作用的描述是这样的：

> Registry instances may use the Redis instance for several applications. Currently, it caches information about immutable blobs. 

先扯了一下淡告诉你docker registry可以利用redis来实现许多功能。（但是）目前仅用来缓存一些不变的blobs信息。OK，不管这些了，我们来配置缓存：

首先，我们在`docker-compose.yml`中加入一个redis service，代码如下：

```YML
redis:
  image: redis:latest # 默认使用的镜像为redis:latest
  command: redis-server --requirepass **redis密码**
  networks:
    - overlaynet # 加入自定义的overlay网络
```

然后我们再去配置registry的环境变量：

```YML
environment:
  # Redis cache
  - REGISTRY_STORAGE_CACHE_BLOBDESCRIPTOR=redis
  - REGISTRY_REDIS_ADDR= # redis服务器地址
  - REGISTRY_REDIS_PASSWORD= # redis 密码
  - REGISTRY_REDIS_DB= # registry所使用的redis数据库编号
  - REGISTRY_REDIS_POOL_MAXIDLE=16 # 连接池-最大空闲连接数
  - REGISTRY_REDIS_POOL_MAXACTIVE=64 # 连接池-最大活动连接数
  - REGISTRY_REDIS_POOL_IDLETIMEOUT=300s # 连接池-空闲超时时间
```

详细的配置可参考其官方文件：

> [redis - Configuring a registry](https://docs.docker.com/registry/configuration/#redis)

### 0x03 nginx代理转发





