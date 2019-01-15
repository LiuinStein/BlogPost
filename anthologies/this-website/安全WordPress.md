# 安全配置日志

## 0x00 前言

目前的这个网站，建立在一个我自认为是相对安全的体系结构下，撰写此篇文章来描绘一下网站的建构

![](https://bucket.shaoqunliu.cn/image/0008.png)

## 0x01 服务器建构

上面的这幅图片描述了网站的大体构造，网站有3台服务器组成，3台服务器架构在2台物理服务器上，分别为

* 网页代码服务器A，仅用来存放和执行网页源代码，代码服务器架构在Docker容器中，宿主机使用CentOS操作系统，Docker虚拟机使用Ubuntu Linux操作系统，宿主机仅运行Docker服务，容器中仅运行Nginx和php-fpm
* 数据库服务器B，仅用来存放数据库，使用mysql数据库，做站库分离，数据库仅允许wordpress用户远程访问，wordpress用户仅对wordpress数据库有增删改查及建表五项权限，对information_schema数据库有usage权限，对其他数据库无访问权限，数据库服务器运行在内网环境，使用内网域名db.shaoqunliu.cn，内网访问端口随机生成
* 对象存储服务器C，仅用来存放网页中静态的图片及文件，对公网暴露80及443端口，使用与代码服务器A对等的CDN配置，与网页代码服务器物理隔离

## 0x02 CDN建构

在互联网领域，有一个著名的8秒定律就是一个网页如果打开的时间超过8秒的话，那么用户点击浏览器的关闭按钮的可能就比较大了，而且Google或百度的SEO对于访问效率也是一个考察的方面

使用CDN结点一是可以提升网站速度，而且可以隐藏服务器真实IP提高网站的安全性

使用CNAME解析配置CDN结点，两台独立服务器均部署单独的CDN结点

## 0x03 服务器安全配置日志 

我使用的是原版的wordpress，安装和配置完成之后，我首先做的就是选择一个好看的主题，然后配置好文章等页面数据以及wordpress插件，等做完这些之后，剩下的就是安全性配置了，下面是一个大体的配置流程

### 0x00 网页代码服务器及数据库服务器环境搭建

我使用的Docker，把网页代码服务器部署在了Docker之上，我设计了几个Docker容器，然后将其commit到了网易蜂巢，网易蜂巢有个Docker的镜像中心在国内的下载速度还是很不错的，如果你想使用国外的Docker Hub上的容器，由于众所周知的原因，是吧服务器的网速不大快，但是可以通过配置DaoCloud上的一个加速器来快速访问，在这里我将我所有的DockerHub上的镜像全部传到了网易蜂巢，我服务器200Mbps的外网带宽，下载可以到10几M每秒

我的网易蜂巢Docker源地址

* Ubuntu16.04+sshd`https://c.163.com/hub#/m/repository/?repoId=43770`	    当前网站代码服务器使用
* Ubuntu16.04+lnmp+sshd`https://c.163.com/hub#/m/repository/?repoId=43783`
* Ubuntu16.04+mysql5.7.16+sshd`https://c.163.com/hub#/m/repository/?repoId=45446`  当前网站数据库使用
* Shadowsocks `https://c.163.com/hub#/m/repository/?repoId=45792`  

这个网站是在这些镜像的基础上改装而来，所有软件全部使用最新版本，安装所有的补丁及升级程序

网站使用的是我在网易蜂巢上上传的Ubuntu16.04+sshd镜像，然后我自己配置了一套nginx和php环境，然后打包了一个私有镜像

Docker环境配置

1. 创建虚拟局域网

   ```shell
   docker network create --subnet=IP/子网掩码 子网名
   ```

   上述命令创建了一个虚拟局域网，IP可以使用以下作为范例172.16.0.0/16，意思是创建一个子网，子网的网段为172.16.0.0，/后面的数字是子网掩码，16就是代表二进制位中有16个1和16个0，也就是255.255.0.0，第一个可用的IP为172.16.0.1

2. 运行网站代码容器

   ```shell
   docker run -d -p 80:80 -p 443:443 -p 某个端口:22 --net 字网名 --ip 固定的IP地址 --name 容器名 镜像名或ID /usr/sbin/sshd -D
   ```

   -d创建了一个守护容器，-p用来做端口映射，--net用来设置子网名，--ip用来设置固定的ip地址，在这里其实是不应该把ssh端口映射出来的，不安全，但是我为了方便在我的电脑上用winscp等工具快速管理服务器我把它映射出来了

3. 运行mysql数据库容器

   ```shell
   docker run -d --net 字网名 --ip 固定的IP地址 --name 容器名 镜像名或ID /usr/sbin/sshd -D
   ```

   mysql服务器不对外暴露端口

4. 然后就可以使用以下三种方式连接和配置服务器

   1. `docker exec -it 容器ID bash` 
   2. `ssh -p 某个端口 root@localhost`
   3. 或者在你的电脑上使用XShell或putty等工具连接


### 0x01 网站目录权限

依照我的习惯，首先通过命令将整个网页目录的所有者和分组改为nobody和nogroup

```shell
chown -R nobody **网页目录**	# 递归设置所有者为nobody
chgrp -R nogroup **网页目录**	# 递归设置分组为nogroup
```

设置完所有者和分组后，接下来就是对网页目录进行权限管控，之前提到过，我们已经安装完wordpress的主题和必要的插件了，wordpress中文章和页面数据都是存放在数据库中的，而不是网页目录中的，所以，通过以下命令将网页目录权限设置为r-x r-x r-x(555)

```shell
chmod -R 555 **网页目录**
```

同时在wp-config.php中追加代码`define('DISALLOW_FILE_EDIT', true);` 禁用文件修改

### 0x02 自定义HTTP错误页面 

nginx自带的错误页面，没一个好看的，自己从网上挑选了一套纯HTML+CSS+JS的错误页面模板，然后分别改造出了7个HTTP错误页面，然后配置/usr/local/nginx/conf/nginx.conf文件，server中添加如下语句

```properties
        error_page   400 =  https://www.shaoqunliu.cn/error/400.html;
        error_page   403 =  https://www.shaoqunliu.cn/error/403.html;
        error_page   404 =  https://www.shaoqunliu.cn/error/404.html;
        error_page   405 =  https://www.shaoqunliu.cn/error/403.html; # (*^__^*) 嘻嘻…… 
        error_page   500 =  https://www.shaoqunliu.cn/error/500.html;
        error_page   502 =  https://www.shaoqunliu.cn/error/502.html;    
        error_page   503 =  https://www.shaoqunliu.cn/error/503.html;  
```

最后

```shell
nginx -t # 检查nginx.conf配置是否正确
nginx -s reload	# 重启nginx
```

### 0x03 精简wordpress

刚安装完的wordpress，在wordpress目录下有好多没必要的文件，在我的理念里，暴露在网页服务器的代码文件越多，存在漏洞的几率就越大，所以对没必要的文件可以进行删除或改名低权，我一般不提倡删除的方式 ，我一般通过改名低权的方式，改名就是把这个文件重命名，低权就是设置为400权限(r-- --- ---)，下面提高了我改造的几个文件

* license.txt	这个是wordpress的开源协议，直接再见
* readme.html  这个是安装wordpress的一个帮助文档，也是再见
* xmlrpc.php   这个是用来接收XML推送文章的，这个功能可以使你不用登录wordpress后台即可发表文章，我之前用过，在MS Office Word2016里面有个分享功能可一键递交至wordpress，我现在一般用Markdown写东西，所以用不着了，同样让他再见
* wp-config-sample.php   用作示例的配置文件，我用不着，再见
* wp-admin/install.php    这个是用来安装wordpress的，安装完成了，自然也不需要了，不过可以通过删除数据库然后调用这个文件进行重装操作，同样再见
* wp-admin/theme-install.php  用于在控制台中安装主题，之前已经安装配置好我喜欢的主题了，不好意思，再见
* wp-admin/plugin-install.php  用于安装wordpress插件，之前已经装好了，同样再见
* wp-admin/plugin-editor.php  用来在控制台页面修改插件，为了防止其中的漏洞可能对插件造成恶意植入，将其删除
* wp-admin/theme-editor.php   用来在控制台页面修改主题代码，为了防止其植入恶意代码将其删除
* wp-admin/user-edit.php    用来在控制台页面对用户信息进行修改，为了防止可能发生的恶意修改将其删除
* wp-admin/user-new.php    用来创建新用户，不用新用户了,将其删除
* wp-admin/setup-config.php    用来进行安装配置，将其删除
* wp-content/languages  不好意思，我只会说中国话，删掉所有不是Zh_CN的翻译
* wp-content/themes    删除所有不用的主题

经过上面的一系列操作，这时我们登录wordpress控制台，我们可以发现很多功能已经报废掉了，比如添加用户，修改插件和添加插件等，如需开放功能可将文件重新部署上即可，我的套路一般是正常运行时报废掉那些文件，需要添加插件的时候仅开放添加插件的文件，添加完成后重新将这些文件重新报废掉

### 0x04 控制台登录的二重验证

在主题的functions.php文件里面添加如下代码，可增加登录界面的二重验证机制

```php
add_action('login_enqueue_scripts','login_protection');
function login_protection() {
	if($_GET['loginhere'] != 'self-string')	// loginhere为验证字段名,self-string为验证字段值
		header('Location: 跳转地址');
}
```

添加了这一小段代码之后，直接访问` https://www.shaoqunliu.cn/wp-admin/`和`https://www.shaoqunliu.cn/wp-login.php`就会自动跳转至预设的跳转地址，正常访问和登录后台需使用链接`https://www.shaoqunliu.cn/wp-login.php?loginhere=self-string` 我对我的网站设置的loginhere参数名为随机字符串，验证字符串为预设的16位随机字符串

### 0x05 使用SSL加密获取gravatar头像 

在主题的functions.php文件里面添加如下代码，可以使用HTTPS加密过去gravatar头像，而且还有过墙（你猜是什么墙）的效果

```php
function get_ssl_avatar($avatar) {
   $avatar = preg_replace('/.*\/avatar\/(.*)\?s=([\d]+)&.*/','<img src="https://secure.gravatar.com/avatar/$1?s=$2&d=mm" class="avatar avatar-$2" height="$2" width="$2">',$avatar);
   return $avatar;
}
add_filter('get_avatar', 'get_ssl_avatar');
```

或者干脆使用以下代码替换为国内的结点(duoshuo.com)

````php
function unblock_gravatar( $avatar ) {
    $avatar = str_replace( array( 'www.gravatar.com', '0.gravatar.com', '1.gravatar.com', '2.gravatar.com' ), 'gravatar.duoshuo.com', $avatar );
    return $avatar;
}
add_filter( 'get_avatar', 'unblock_gravatar' );
````

### 0x06 修改管理员默认名称

安装的时候我用的默认的admin作为管理员的默认名称，通过修改数据库的方式将其修改为自定义的名字，在数据库中使用以下语句

```sql
update wp_users set user_login='登录名', user_nicename='昵称' where id=1; # 呀,博客泄露表名和字段名了
```

### 0x07 去除wordpress版本信息

wordpress有个毛病就是喜欢在head标签中输出一个meta，里面包含着wordpress的版本信息，就像下面一样

```html
<meta name="generator" content="WordPress 4.6">	<!-- Wordpress版本号泄露 -->
```

而且某些CSS和js文件结尾也是喜欢输出版本号

```html
<link rel='stylesheet' id='dashicons-css'  href='https://www.shaoqunliu.cn/wordpress/wp-includes/css/dashicons.min.css?ver=4.6' type='text/css' media='all' />
<link rel='stylesheet' id='admin-bar-css'  href='https://www.shaoqunliu.cn/wordpress/wp-includes/css/admin-bar.min.css?ver=4.6' type='text/css' media='all' />
```

在主题functions.php文件里面添加如下代码去除head，feed以及js/css中的wordpress版本号

```php
// 同时删除head和feed中的WP版本号
function remove_wp_version() {
	return '';
}
add_filter('the_generator', 'remove_wp_version');
// 隐藏js/css附加的WP版本号
function remove_wp_version_strings( $src ) {
	global $wp_version;
	parse_str(parse_url($src, PHP_URL_QUERY), $query);
	if ( !empty($query['ver']) && $query['ver'] === $wp_version ) {
		// 用WP版本号 + 12.11来替代js/css附加的版本号
		// 既隐藏了WordPress版本号，也不会影响缓存
		// 不要问我为啥是1211,我媳妇生日,嘻嘻(*^__^*) 嘻嘻……
		$src = str_replace($wp_version, $wp_version + 12.11, $src);
	}
	return $src;
}
add_filter( 'script_loader_src', 'remove_wp_version_strings' );
add_filter( 'style_loader_src', 'remove_wp_version_strings' );
```

因为禁用了对外注册和创建其他用户，所以不用担心wordpress后台控制面板的版本号，如果有强迫症非要去除那段版本号的话，可以在functions.php中追加以下代码

```php
add_filter('admin_footer_text', 'left_admin_footer_text');
function left_admin_footer_text($text) {
  // 左边信息改成自己的站点
  $text = '感谢访问XXXX';	// 默认左边那句"感谢使用WordPress进行创作"
  return $text;
}

add_filter('update_footer', 'right_admin_footer_text', 11);
function right_admin_footer_text($text) {
  // 隐藏右边版本信息
}
```

### 0x08 全站HTTPS访问

众所周知HTTP在传输的过程中是明文传输，容易遭到中间人的一些操作，HTTPS协议是加密传输，HTTPS使用认证过的SSL证书对传输内容进行加密，而且搜索引擎SEO也偏重HTTPS网站，我的网站使用TrustAsia Inc颁发的SSL证书，下面简要介绍一下证书的部署流程

1. 首先向证书认证机构(CA)申请一个证书，然后下载这个证书，证书包含两个文件，一个是crt文件，一个是key文件，key文件中包含一个RSA私钥，大致的传输原理流程如下，服务器 用RSA生成公钥和私钥，把公钥放在证书里发送给客户端，私钥自己保存，客户端首先向一个权威的服务器（CA）检查证书的合法性，如果证书合法，客户端产生一段随机数，这个随机数就作为通信的密钥，我们称之为对称密钥，用公钥加密这段随机数，然后发送到服务器，服务器用私钥解密获取对称密钥，然后，双方就已对称密钥进行加密解密通信了

2. 有了这两个文件之后将其放在服务器上一个相对安全的位置，然后配置/usr/local/nginx/conf/nginx.conf文件

   ```properties
   *****隐藏部分*****
   	server
       {	# 对http请求301(永久重定向)到https请求
               listen  80;	
               server_name www.shaoqunliu.cn;
               return 301 https://www.shaoqunliu.cn$request_uri;
       }
       
       server
       {
           listen 443 ssl;		# 监听443端口,HTTPS默认端口
           server_name www.shaoqunliu.cn;	# 服务器名
           index index.html index.htm index.php default.html default.htm default.php;	# 索引顺序
           root  ***网页目录***;
           ssl on;
           ssl_certificate ***证书crt文件所在地***;
           ssl_certificate_key ***证书key文件所在地***;
           ssl_session_timeout 5m;
           ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
           ssl_prefer_server_ciphers on;
           ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-RC4-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:RC4-SHA:!aNULL:!eNULL:!EXPORT:!DES:!3DES:!MD5:!DSS:!PKS;	# 支持的加解密算法
           ssl_session_cache builtin:1000 shared:SSL:10m;
   *****隐藏部分*****
   ```

   然后使用`nginx -t` 检查配置文件是否正确，正确则使用`nginx -s reload` 来重启nginx服务器

3. 在wordpress中对wp-admin/控制面板和wp-login.php登录页面做强制SSL访问，在wp-config.php中追加以下代码

   ```php
   define('FORCE_SSL_ADMIN', true);
   define('FORCE_SSL_LOGIN', true);
   ```

### 0x09 数据库安全

本站中数据库使用独立的服务器，由网页代码服务器向内网独立数据库服务器获取数据，在访问过程中使用以下语句来创建访问用户

```sql
GRANT SELECT, INSERT, DELETE, UPDATE, CREATE ON wordpress使用的数据库名称.* TO '数据库用户名'@'主机名' Identified by '随机密码';
flush privileges;
```

上述语句的第一条为创建用户，这个用户用户名在这里记做user，user用户仅对wordpress所使用的数据库中的全部的表有查找，插入，删除，更新以及创建表这五项操作的权利，在@后面限制主机名，即使在内网环境中也仅有对应主机名的服务器可以访问数据库

这个用户对mysql中其余的数据库均没有任何的权限

同时内网中设定域名解析，设置域名db.shaoqunliu.cn解析到数据库服务器的内网ip地址，wordpress配置中使用db.shaoqunliu.cn隐藏内网服务器IP

修改mysql配置文件中的端口号为随机端口来进一步提升安全性

在网页服务器中管理数据库可使用以下语句获取mysql控制台(低权)

````shell
mysql -h db.shaoqunliu.cn -P 随机端口号 -uwordpress数据库用户名 -p
````

需要有较高权限的进一步配置需用内网ssh登录服务器

```shell
ssh -p 数据库服务器SSH端口号 root@db.shaoqunliu.cn
```

然后使用以下语句以root权限登录服务器

```shell
mysql -uroot -p	
```

### 0x0A 上传目录安全

wordpress有一个上传目录，主要用来保存一些用户上传的媒体和资料文件，这个文件夹一旦被上传了一句话就是一个比较尴尬的事了，通过配置nginx配置文件来禁用上传目录php文件的执行，如果黑客上传的文件为Php或者PHp这样后缀变化过的，那么其压根不会被解析执行，因为之前设置的解析到php-fpm就是只解析php文件，使用Php访问就会下载这个php文件

```properties
        location ~ /wp-content/uploads/.*\.php$ 
        { 
            deny all; 
        }
```

这样即便是一句话上传成功也没有办法使其执行，返回的是一个403禁止访问的错误，其实一开始的时候已经将所有的写权限全部去掉了，要想上传文件的话，需要用SSH登录服务器然后才能上传文件，在这里只是一个备用

### 0x0B 禁止直接通过服务器IP访问

虽然说之前用了CDN，但是有的时候我老是害怕有些黑客它就和你刚上了，它和你刚上了，他会怎么办呢，他用服务器架构ZMAP然后跑全网，全球一共有42亿个IPv4的地址，就是256的四次方，去掉国外的，去掉局域网内的，然后到咱中国的IP可能也就3,4亿个(我知道有个Asia表里面有全亚洲的IP地址集合, China表应该也有单独的)，zmap可以在10Gbps的宽带情况下，用45min跑遍全球42亿个IP地址，这个值是他们官方给出的，我自己并没有试过，因为我一直没有一台可以达到10Gbps宽带的服务器，我现在用的这个才200Mbps，10Gbps的50分之一，我就害怕他们和我刚上，然后跑遍全网，找到我的真实IP，然后把我打得不要不要的

于是我就通过配置nginx，实现使用ip访问网站nginx爆出400错误，nginx配置如下

```properties
    server  
    {  
    	# 这是nginx官方推荐的方法
        listen 80 default;  
        return 400;  	# 返回400错误
    } 
```

这样在浏览器里面输入真实ip然后访问域名就会看到一个nginx默认的400错误页面

但是后来我一想如果通过https://服务器真实IP的形式仍然可以访问到网站啊，这个配置对HTTPS不起作用，而且返回400错误也不大友好不是，我尝试了以下配置

```properties
    server  
    {   
        listen 443 default;  
        return 400;  	# 返回400错误
    } 
```

这个配置上去之后，连正常的通过https域名访问都不可以了，肯定是不行的，那怎么办呢，于是我想到了rewrite这个神奇的关键字，rewrite可以将请求移走，那不如我们给百度贡献一点流量，重新配置nginx如下

```properties
****隐藏部分****
	server  
    {  	
    	# 这是nginx官方推荐的方法,仅适用于80端口
        listen 80 default; 
        rewrite ^(.*) http://www.baidu.com permanent;  # permanent设置永久性
    }  
    
    server
    {
        listen 443 ssl;
        # 443端口方案1
        if ($host ~ "服务器真实IP,比如123.123.123.123") {
            rewrite ^(.*) http://www.baidu.com permanent;	# 我认为百度老总一定会很感激我的
        }
        # 443端口方案2(推荐),只允许使用域名访问
        if ( $host !~* 'shaoqunliu.cn' ) {  
            rewrite ^(.*) http://www.baidu.com permanent;   
        }  
****隐藏部分****
```

经过上述配置之后，不管是使用http://服务器真实IP还是使用https://服务器真实ip，都被重写到了百度的页面，给百度贡献一点流量

### 0x0C 伪静态

伪静态在严格意义上来说应该不算是安全方面的配置吧，但有的时候隐藏后端语言是一个不错的方法，因为我都公开了我的blog是用wordpress做的，基本上是个人就应该知道wordpress是用php写出来的一套开源的东西，但我还是做了伪静态

配置nginx

```properties
        location / {
            try_files $uri $uri/ /wordpress/index.php?$args;
        }

        rewrite /wp-admin$ $scheme://$host$uri/ permanent;  
```

try_files用来判断一个资源是否存在，有则返回，没有则交由/wordpress/index.php作为一个args(参数)处理，然后配置自定义结构为`https://www.shaoqunliu.cn/%post_id%.html`，这样伪静态就做完了

### 0x0D 规避php和nginx搭配时的解析漏洞

这个漏洞只存在于php和nginx的搭配环境中(不是nginx本身的原因，是配置问题，正则表达式没匹配好)，php和Apache等服务器搭配均没有这个问题，问题的起源就是，黑客构造一个带有恶意代码的a.jpg上传到你的服务器上去，然后通过类似于以下的方式`https://www.shaoqunliu.cn/a.jpg/hacker.php` 然后就可以将a.jpg中的恶意代码暴露出来，触发原理就是使得php触发逻辑将a.jpg解析为filename，将hacker.php解析为pathinfo，从而把a.jpg当做php文件来执行，修改php.ini文件，查找并设置以下参数

```properties
cgi.fix_pathinfo=0		# 设置此参数为0可规避以上漏洞
```

### 0x0E 限制php文件访问的范围

这个一开始我设置了，但是有一个取舍就是性能，想要安全还是想要性能

```properties
open_basedir = 网页目录，例如/var/www/
```

设置了这个参数之后，每一次include，require请求都会判断请求的文件是不是在这个open_basedir目录下，对性能有一定的损失，当然设置了之后黑客使用的php木马中的fopen打开文件等函数统统限制在了open_basedir目录下

### 0x0F 限制执行php部分危险函数

这个参数不用我介绍了吧

```properties
disable_functions = rename,fputs,fwrite,delete,copy,dir,unlink,chroot,phpinfo,exec,passthru,openlog,syslog,shell_exec,system,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source,chgrp,chown,getcwd,rmdir
```

你知道改这个参数最日狗的是事是什么吗，改了之后啊，你wordpress里面插件不能用了，主题加载不出来了，然后不得不使用排除法，又去掉了一些wordpress所必须的函数，才搞好的

之前有个哥们问我在这里限制eval函数为什么不行啊，这肯定是不行的，eval不是php的函数，eval是zend的东西，它并不是一个php函数，所以使用disable_functions限制是不管用的，如果想限制eval函数的执行的话，需要安装插件Suhosin，然后使用`suhosin.executor.disable_eval = on` 禁用eval函数，配置Suhosin我放在下一节，eval函数在某些场合是要用到的，如果禁用了eval发现一些插件的功能不能用了，请关闭这项功能，原版的wordpress不需要eval，我查过StackExchange（http://wordpress.stackexchange.com/questions/246251/does-wordpress-need-the-eval-function）

### 0x10 php安全插件Suhosin

Suhosin是韩国棒子开发的一款php安全插件，这个单词就是音译的韩语里面守护神的发音，其实我们汉语的守护神发音音译过去也差不多吧，而且在他们的官网，还有一个守护神的图片，那个图片明明是我们中国的一个神像，可耻的韩国棒子又说这是他们韩国的东西了，真欠揍我看他们。

好了说完不要脸的韩国棒子，再来说说这个插件，这个插件可以提供php内核级的保护，而且实现了好多有用的功能，比如加密cookie，加密session，防止恶意的session_id，以及抵御缓冲区溢出，防止破坏zend哈希表等功能（后面的两个我并没有自己用过）

```shell
wget -c https://download.suhosin.org/suhosin-0.9.38.tar.gz	# 截至Dec 25,2016的最新版(稳定版)
tar zxvf suhosin-0.9.38.tar.gz	# 解压
cd suhosin-0.9.38
/usr/local/php/bin/phpize	# 生成configure配置文件
./configure --with-php-config=/usr/local/php/bin/php-config	# 设置参数
make && make install	# 编译安装
```

然后将以下代码放进php.ini，然后就可以进行配置了，官方的手册在这里`https://suhosin.org/stories/configuration.html` ，我就是比着这手册配出来的

```properties
[suhosin]
extension=suhosin.so
suhosin.get.max_value_length = 5120

;; 只允许权限为只读的php文件执行
suhosin.executor.include.allow_writable_files = Off

;; 禁用函数
suhosin.executor.disable_eval = On
suhosin.executor.func.blacklist = assert
suhosin.executor.disable_emodifier = On

;; 加密cookie
suhosin.cookie.encrypt = On

;; 加密密钥
suhosin.cookie.cryptkey = 32位的加密密钥 
suhosin.cookie.cryptua = On
suhosin.cookie.cryptdocroot = On
```

`suhosin.executor.include.allow_writable_files = Off` 限制了只有php文件为只读的属性的时候才允许其执行，上传的一句话木马是具有写权限的，但是我一般设置的wordpress都是555的权限（r-x r-x r-x），通过设置这个选项意味着带带写权限的php文件无法被执行

`suhosin.executor.disable_eval = On` 禁用eval函数

`suhosin.executor.func.blacklist = assert` 禁用assert函数

`suhosin.executor.disable_emodifier = On` 禁止函数 preg_replace() 在/e模式下运行

`suhosin.cookie.encrypt = On` 加密cookie

`suhosin.cookie.cryptkey = 32位的加密密钥` 设置cookie加密时的密钥

`suhosin.cookie.cryptua = On` 用设置好的密钥加密UA(User-Agent)

`suhosin.cookie.cryptdocroot = On` 设置固定的加密密钥，就是上面指定的那个参数

然后就是重启php-fpm了`service php-fpm restart`

通过上传实验性的带eval的一句话木马，然后使用菜刀连接就会返回500服务器错误的信息

### 0x11 杂项

隐藏X-Powered-By参数

有的时候吧，在Http请求里面返回的数据包中就有这样的一个字段就是X-Powered-By，这个字段里面有服务端使用的php的详尽版本，就像下面这样`X-Powered-By: PHP/5.2.13`，这可不好

```properties
expose_php = Off	# 这个也能隐藏php彩蛋,具体什么是php彩蛋,你猜
```

禁止打开远程文件和包含远程文件

```properties
allow_url_fopen = Off
allow_url_include = Off
```

### 0x12 生效及容器备份

需要上面的配置生效需要使用以下命令

```shell
root@2022ff812a2c:~# nginx -t	# 检查nginx配置文件是否正确
nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
root@2022ff812a2c:~# nginx -s reload	# 重新载入nginx
root@2022ff812a2c:~# service php-fpm restart	# 重启php-fpm
Gracefully shutting down php-fpm . done
Starting php-fpm  done
root@2022ff812a2c:~# php -v
root@2022ff812a2c:~# history -c	# 清理bash日志
PHP *隐藏版本号* (cli) (built: Dec  7 2016 21:28:53) 
Copyright (c) 1997-2016 The PHP Group
Zend Engine *隐藏版本号*, Copyright (c) 1998-2016 Zend Technologies
    with Zend Guard Loader *隐藏版本号*, Copyright (c) 1998-2014, by Zend Technologies
    with Suhosin *隐藏版本号*, Copyright (c) 2007-2015, by SektionEins GmbH
    # 看到with Suhosin就说明守护神已经生效了
```

备份容器，以便黑客攻击之后快速恢复

```shell
[root@**隐藏部分** ~]# docker ps
CONTAINER ID        IMAGE               COMMAND               CREATED             STATUS              PORTS                                                             NAMES
bfb5c28eb2a4        36971               "**隐藏部分**"   2 days ago          Up 2 days           22/tcp, 3306/tcp                                                  blogdb
2022ff812a2c        ffd76db3            "**隐藏部分**"   2 days ago          Up 2 days           0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:**隐藏部分**->22/tcp   blogweb
[root@**隐藏部分** ~]# docker commit bfb5 dbbakup-12-25	# 备份容器
sha256:d133fd47485687aa9bf8c441a9919e158b779a17105da64f90f0b3298baf83cc
[root@**隐藏部分** ~]# docker commit 2022 blogbakup-12-25 # 备份容器
sha256:54da44f8e38ef6769cd337375e74e22be6b29511c4c80615aad5218878a552d2
[root@**隐藏部分** ~]# docker images
REPOSITORY                              TAG                 IMAGE ID            CREATED             SIZE
blogbakup-12-25                         latest              54da44f8e38e        4 seconds ago       2.843 GB
dbbakup-12-25                           latest              d133fd474856        38 seconds ago      1.058 GB
database                                latest              36971321c0a7        3 days ago          895.2 MB
blogbak-12-21                           latest              ffd76db3beb7        3 days ago          2.801 GB
hub.c.163.com/liuinstein/mysql-sshd     latest              1e51950ae35f        3 days ago          658.6 MB
hub.c.163.com/liuinstein/shadowsocks    latest              f091cf0edcb4        5 days ago          52.56 MB
hub.c.163.com/liuinstein/ubuntu-lnmp    latest              aa76f29525ea        2 weeks ago         2.611 GB
hub.c.163.com/liuinstein/sshd           latest              ebdbd0de15bb        2 weeks ago         222.6 MB
daocloud.io/daocloud/daocloud-toolset   latest              1ab33797d8a1        8 months ago        150.2 MB
```

### 0x13 安全组配置及SSH密钥登录

安全组起到各司其职的作用，日常情况下仅放通服务器的80和443端口，如需管理服务器依照管理需求进行安全组之间的切换，例如，如需管理Docker宿主机进行容器备份风操作，则切换为安全组sg-1w7h4ntr，安全组仅增加放通用于Docker宿主机管理使用的SSH登陆端口，操作完成之后将安全组切换回日常使用的安全组

使用密码加密过的SSH密钥登录服务器，每个密钥仅能绑定一台服务器

### 0x14 安全测试(非渗透)

使用winscp以合法的身份上传一句话木马测试是否执行，测试文件权限r-x r-x r-x，测试文件夹允许php文件执行

````php
// 常见一句话木马测试
<?php @eval($_POST['pwd']); ?>	// 服务器返回500错误
<?php @assert($_POST['pwd']);?> // 服务器不返回任何数据
<?php @$_++;$__=("#"^"|").("."^"~").("/"^"`").("|"^"/").("{"^"/");@${$__}[!$_](${$__}[$_]);?>	// 服务器不返回任何数据,上述代码实质: $_POST[0]($_POST[1]);
<?php @preg_replace("//e",$_POST['cmd'],"");?>  // 服务器返回500错误
````

Webshell可用性测试

![](https://bucket.shaoqunliu.cn/image/0012.png)

![](https://bucket.shaoqunliu.cn/image/0013.png)

![](https://bucket.shaoqunliu.cn/image/0014.png)

![](https://bucket.shaoqunliu.cn/image/0015.png)

![](https://bucket.shaoqunliu.cn/image/0016.png)

同时，经测试phpspy因为服务器直接返回500错误死掉

下面的合法的代码因php文件被赋予了写权限而直接死掉，测试权限r-x r-x rwx，仅其他被赋予写权限，使用r-x r-x r-x权限可正常执行

```php
<?php echo '123'; ?>
```

Burpsuite抓包见以下cookie，加密后的cookie

```properties
wp-settings-1=UNJX8h-IPRHmng_nOPaAp-4bTDIua9z2392Etp2Y0I7KEnLwfJQz-qx4E17CzmGFVpOUQoJUD9sDnaeFT9-iZA..; wp-settings-time-1=MYQrohMog2ScOz-QZJbQUev2pADyj4tWxyE0mmFU0Z0.
```

配置cookie加密之前抓下来的用于比对的包内容，加密前的cookie

```properties
wp-settings-1=libraryContent%3Dbrowse%26editor%3Dtinymce%26mfold%3Do; wp-settings-time-1=1482382409;
```

