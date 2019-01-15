### 0x00 前言

之前的服务器一直使用的lnmp自动化安装脚本来安装nginx, php-fpm等服务器环境，lnmp脚本有个缺点就是你可以一次性安装nginx, php-fpm, MySQL，但是你不能去分别安装这些服务，而且nginx貌似还不能自己加模块的样子。所以，喜欢瞎折腾的我边学边试了一把自己编译安装这些环境，记录一下，以便备忘。

### 0x01 编译安装Nginx

##### 安装所必需的依赖：

```shell
apt-get update && apt-get upgrade	# 更新源和升级软件
apt-get install gcc g++ make openssl libssl-dev libpcre3-dev # 安装必需的依赖
```

##### 下载Nginx及ngx_cache_purge缓存模块（非必需）：

```shell
cd ~	# 切换目录
# 下载并解压nginx-1.10.3
wget http://nginx.org/download/nginx-1.10.3.tar.gz
tar zxvf nginx-1.10.3.tar.gz
cd nginx-1.10.3
# 下载并解压ngx_cache_purge（非必需）
wget http://labs.frickle.com/files/ngx_cache_purge-2.3.tar.gz
tar zxvf ngx_cache_purge-2.3.tar.gz
cd ..
```

##### 添加Nginx运行时所使用的用户和用户组（如果你想使用现有的用户，此步可忽略）：

```shell
groupadd -f www		
useradd -g www www
```

##### 检查配置：

```sh
./configure \
--user=www \
--group=www \
--prefix=/usr/local/nginx \
--pid-path=/var/run/nginx.pid \
--lock-path=/var/lock/nginx.lock \
--with-http_stub_status_module \
--with-http_ssl_module \
--with-http_gzip_static_module \
--with-http_v2_module \
--with-http_sub_module \
--add-module=ngx_cache_purge-2.3 
```

如果它提示缺少什么模块的话，那就可以使用如下命令解决

```shell
apt-get install lib缺少的模块名-dev	# 例如libssl-dev
```

##### 编译安装：

检查配置无误之后就可以编译安装了

```shell
make && make install
```

##### 添加服务

添加服务之后，我们就可以方便地使用`service`来进行重启，关闭等操作了

执行命令：

```shell
vi /etc/init.d/nginx
```

然后填写以下内容：

```sh
#! /bin/sh

### BEGIN INIT INFO
# Provides:          nginx
# Required-Start:    $local_fs $remote_fs $network $syslog
# Required-Stop:     $local_fs $remote_fs $network $syslog
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: starts the nginx web server
# Description:       starts nginx using start-stop-daemon
### END INIT INFO

PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
DAEMON=/usr/local/nginx/sbin/nginx
NAME=nginx
DESC=nginx

DAEMON_OPTS=''

test -x $DAEMON || exit 0

# Include nginx defaults if available
#if [ -f /etc/default/nginx ] ; then
#       . /etc/default/nginx
#fi

set -e

. /lib/lsb/init-functions

#test_nginx_config() {
#  if $DAEMON -t $DAEMON_OPTS
#  then
#    return 0
#  else
#    return $?
#  fi
#}

case "$1" in
  start)
        echo -n "Starting $DESC: "
        start-stop-daemon --start --quiet --pidfile /var/run/$NAME.pid \
                --exec $DAEMON || true
        echo "$NAME."
        ;;
  stop)
        echo -n "Stopping $DESC: "
        start-stop-daemon --stop --quiet --pidfile /var/run/$NAME.pid \
                --exec $DAEMON || true
        echo "$NAME."
        ;;
  restart|force-reload)
        echo -n "Restarting $DESC: "
        start-stop-daemon --stop --quiet --pidfile \
                /var/run/$NAME.pid --exec $DAEMON || true
        sleep 1
        start-stop-daemon --start --quiet --pidfile \
                /var/run/$NAME.pid --exec $DAEMON || true
        echo "$NAME."
        ;;
  reload)
        echo -n "Reloading $DESC configuration: "
        start-stop-daemon --stop --signal HUP --quiet --pidfile /var/run/$NAME.pid \
            --exec $DAEMON || true
        echo "$NAME."
        ;;
  status)
        status_of_proc -p /var/run/$NAME.pid "$DAEMON" nginx && exit 0 || exit $?
        ;;
  *)
        echo "Usage: $NAME {start|stop|restart|reload|force-reload|status}" >&2
        exit 1
        ;;
esac

exit 0
```

##### 添加权限

```shell
chmod +x /etc/init.d/nginx
update-rc.d nginx defaults
```

ln -s /usr/local/nginx/sbin/nginx /usr/local/sbin

##### 添加软连接

```shell
ln -s /usr/local/nginx/sbin/nginx /usr/local/sbin
```

添加了软连接之后，我们就可以在shell中直接使用nginx命令了

### 0x02 参考资料

[php-fpm and nginx init.d script](https://gist.github.com/pzorn/1762357)