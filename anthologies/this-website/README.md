### 0x00 网站第一版

网站第一版基于[Godaddy](https://sg.godaddy.com/zh/websites/wordpress)的WordPress建站主机，购买之后，你能拿到的只有一个WordPress的后台，你只需要将你自己的域名解析到他给定的一个IP即可实现，这个玩意有且只有一个好处就是服务端由它来管理，你就不用操心服务器的安全性、WordPress及其插件的升级等事宜，它会自动在你做。但是这玩意坏处却一大堆，第一、不自由，你很难对WordPress进行代码上的修改，而且它对此方面进行的限制较多。第二、美国的IP在中国访问慢，且无相关的国内CDN服务。第三、你对于实体的服务器是没有任何控制权的，这样就丧失了学习很多其他知识的机会。

### 0x01 网站第二版

网站第二版截图留念：

![](https://bucket.shaoqunliu.cn/image/0267.jpg)

第二版就购入了一台VPS主机，然后进行搭建。第二版其实可以分2.1和2.2两个版本，初次建站从零开始配置，安全意识较差，导致2.1版被黑掉了一次，所以2.2版对其加强了安全配置。因此当时就撰写了一些有关WordPress安全的文稿，收录如下：

[WordPress安全配置](/anthologies/this-website/安全WordPress) 本文共7682字，阅读大约需要32分钟

> 本文详细介绍了基于PHP的WordPress文档程序的安全配置，其中包括了：站库分离、动静分离、CDN网络配置、WordPress控制台登陆的二重验证、SSL证书配置、数据库安全配置、禁止IP直接访问、动态网页的伪静态、php和nginx搭配时的URI解析漏洞的规避、限制php的文件访问行为以及限制php的部分危险函数执行、php的安全插件Suhosin的使用，以及部分安全测试的结果

[编译安装nginx](/anthologies/this-website/编译安装nginx) 本文共705字，阅读大约需要3分钟

> 本文讲述了从源代码开始编译安装nginx的全过程，以及ngx_cache_purge缓存模块的安装过程

[编译安装php-fpm](/anthologies/this-website/编译安装php-fpm) 本文共781字，阅读大约需要3分钟

> 本文讲述了从源代码开始编译安装php-7的全过程

[配置SSH登陆](/anthologies/this-website/配置SSH登陆) 本文共582字，阅读大约需要2分钟

> 本文详细讲述了使用公私钥的免密码SSH登陆机制，讲述了在Linux下使用`ssh-keygen`以及在Windows下使用`puttygen`进行密钥的生成

### 0x02 网站第三版

网站第三版就是现在你看到的这一版，使用基于`nodejs`的`docsify`搭建而成，静态网站，没有数据库，而且速度也不错。评论系统使用了第三方的`disqus`，目前这个东西在国内好像要翻墙才能用，谁知道呢。

[对富js网站的一点思考](/anthologies/this-website/对富js网站的一点思考) 本文共2145字，阅读大约需要9分钟

> 富js（JavaScript Rich）网站对CDN不友好，而且对SEO不友好。本文主要从这两方面入手，来对此类网站的建设进行思考。主要讨论了世界各大搜索引擎对js客户端渲染的支持程度，以及Google的Caffeine引擎对富js网站的索引过程和相关SEO优化方案。

!> 因为目前在用，生产环境的一些秘密和部分安全配置不能被公开，请见谅

### 0x03 CTF writeup

[CTF-penetration-writeup-1](/anthologies/this-website/ctf-penetration-writeup-1) 本文共3848字，阅读大约需要16分钟

> 2016年11月份山东理工大学网络安全协会CTF比赛，一时兴起，为其命制了一道渗透测试的题目，我水平较差而且不会出题，有些题在设置上不是很合理，这个仅供参考，留一个参考在这。