### 0x00 前言

之前的服务器一直使用的lnmp自动化安装脚本来安装nginx, php-fpm等服务器环境，lnmp脚本有个缺点就是你可以一次性安装nginx, php-fpm, MySQL，但是你不能去分别安装这些服务，而且nginx貌似还不能自己加模块的样子。所以，喜欢瞎折腾的我边学边试了一把自己编译安装这些环境，记录一下，以便备忘。

### 0x01 编译安装php-fpm

因为从网上看到php7的执行效率比php5高了一倍的消息，所以给我的网站马上换上了php7，在这里编译安装的是php7.1.3

#### 下载并解压php7

```shell
wget http://cn2.php.net/get/php-7.1.3.tar.bz2/from/this/mirror
tar xvf mirror && cd php-7.1.3
```

#### 安装所必须的依赖

```shell
apt-get install libkrb5-dev \
libc-client2007e                 \
libc-client2007e-dev             \
libcurl4-openssl-dev             \
libbz2-dev                       \
libjpeg-dev                      \
libmcrypt-dev                    \
libxslt1-dev                     \
libxslt1.1                       \
libpq-dev                        \
libpng12-dev                     \
libfreetype6-dev                 \
build-essential                  \
git                                \
libopenssl-dev
```

#### 为libssl创建软连接

```shell
ln -s /usr/lib/x86_64-linux-gnu/libssl.so  /usr/lib
```

#### 检查配置

```shell
./configure \
--prefix=/usr/local/php                      \
--with-config-file-path=/usr/local/php/etc   \
--with-zlib-dir                              \
--with-freetype-dir                          \
--enable-mbstring                            \
--with-libxml-dir=/usr                       \
--enable-soap                                \
--enable-calendar                            \
--with-curl                                  \
--with-mcrypt                                \
--with-zlib                                  \
--with-gd                                    \
--disable-rpath                              \
--enable-inline-optimization                 \
--with-bz2                                   \
--with-zlib                                  \
--enable-sockets                             \
--enable-sysvsem                             \
--enable-sysvshm                             \
--enable-pcntl                               \
--enable-mbregex                             \
--enable-exif                                \
--enable-bcmath                              \
--with-mhash                                 \
--enable-zip                                 \
--with-pcre-regex                            \
--with-pdo-mysql                             \
--with-mysqli                                \
--with-mysql-sock=/var/run/mysqld/mysqld.sock \
--with-jpeg-dir=/usr                         \
--with-png-dir=/usr                          \
--enable-gd-native-ttf                       \
--with-openssl                               \
--with-fpm-user=www-data                     \
--with-fpm-group=www-data                    \
--enable-ftp                                 \
--with-imap                                  \
--with-imap-ssl                              \
--with-kerberos                              \
--with-gettext                               \
--with-xmlrpc                                \
--with-xsl                                   \
--enable-opcache                             \
--enable-fpm     
```

跟编译安装Nginx一样，如果它提示缺少什么模块的话，那就可以使用如下命令解决

```shell
apt-get install lib缺少的模块名-dev	# 例如libssl-dev
```

#### 编译安装

```shell
make && make install
```

#### 创建软连接

```shell
ln -sf /usr/local/php/sbin/php-fpm /usr/bin/php
```

创建软连接之后我们就可以在shell里使用php命令来直接执行了

#### 设置配置文件

在这里直接将默认的配置文件拷贝了一下

```shell
mv /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
mv /usr/local/php/etc/php-fpm.d/www.conf.default /usr/local/php/etc/php-fpm.d/www.conf
cp ./php.ini-production /usr/local/php/etc/php.ini
```

#### 添加服务

添加服务之后，我们就可以使用service命令来方便管理了：

```shell
vi /etc/init.d/php-fpm
```

添加如下代码：

```shell
#! /bin/sh

### BEGIN INIT INFO
# Provides:          php-fpm
# Required-Start:    $remote_fs $network
# Required-Stop:     $remote_fs $network
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: starts php-fpm
# Description:       starts the PHP FastCGI Process Manager daemon
### END INIT INFO

prefix=/usr
exec_prefix=/usr

php_fpm_BIN=/usr/local/php/sbin/php-fpm
php_fpm_CONF=/usr/local/php/etc/php-fpm.conf
php_fpm_PID=/usr/local/php/var/run/php-fpm.pid


php_opts="--fpm-config $php_fpm_CONF"


wait_for_pid () {
        try=0

        while test $try -lt 35 ; do

                case "$1" in
                        'created')
                        if [ -f "$2" ] ; then
                                try=''
                                break
                        fi
                        ;;

                        'removed')
                        if [ ! -f "$2" ] ; then
                                try=''
                                break
                        fi
                        ;;
                esac

                echo -n .
                try=`expr $try + 1`
                sleep 1

        done

}


case "$1" in
        start)
                echo -n "Starting php-fpm "

                $php_fpm_BIN $php_opts

                if [ "$?" != 0 ] ; then
                        echo " failed"
                        exit 1
                fi

                wait_for_pid created $php_fpm_PID

                if [ -n "$try" ] ; then
                        echo " failed"
                        exit 1
                else
                        chmod 666 /run/php-fpm.sock
                        echo " done"
                fi
        ;;

        stop)
                echo -n "Gracefully shutting down php-fpm "

                if [ ! -r $php_fpm_PID ] ; then
                        echo "warning, no pid file found - php-fpm is not running ?"
                        exit 1
                fi

                kill -QUIT `cat $php_fpm_PID`

                wait_for_pid removed $php_fpm_PID

                if [ -n "$try" ] ; then
                        echo " failed. Use force-quit"
                        exit 1
                else
                        echo " done"
                fi
        ;;

        force-quit)
                echo -n "Terminating php-fpm "

                if [ ! -r $php_fpm_PID ] ; then
                        echo "warning, no pid file found - php-fpm is not running ?"
                        exit 1
                fi

                kill -TERM `cat $php_fpm_PID`

                wait_for_pid removed $php_fpm_PID

                if [ -n "$try" ] ; then
                        echo " failed"
                        exit 1
                else
                        echo " done"
                fi
        ;;

        restart)
                $0 stop
                $0 start
        ;;

        reload)

                echo -n "Reload service php-fpm "

                if [ ! -r $php_fpm_PID ] ; then
                        echo "warning, no pid file found - php-fpm is not running ?"
                        exit 1
                fi

                kill -USR2 `cat $php_fpm_PID`
                chmod 666 /run/php-fpm.sock
                echo " done"
        ;;

        *)
                echo "Usage: $0 {start|stop|force-quit|restart|reload}"
                exit 1
        ;;

esac

```

### 0x02 参考资料

[php-fpm and nginx init.d script](https://www.shaoqunliu.cn/link/aHR0cHM6Ly9naXN0LmdpdGh1Yi5jb20vcHpvcm4vMTc2MjM1Nw==)