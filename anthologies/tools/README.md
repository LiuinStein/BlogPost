### 0x00 索引目录

#### 0x00 Docker三剑客

docker三剑客一般指docker, docker-compose, docker-machine三种工具。



#### 0x01 Git

下文基于17年3月份我在实验室内进行技术报告的PPT汇编整理而成。

[为什么要进行版本控制](/anthologies/tools/git/为什么要进行版本控制) 本文共446字，阅读大约需要2分钟

> 本文主要介绍了版本控制的理由和好处

[基于命令行的Git](/anthologies/tools/git/基于命令行的Git) 本文共1842字，阅读大约需要8分钟

> 本文**转载自**阮一峰博客[常用 Git 命令清单](http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)
>
> 主要介绍了Git命令行工具的使用及相关命令

[在VS中使用GitHub](/anthologies/tools/git/在VS中使用GitHub) 本文共3041字，阅读大约需要13分钟

> 本文详细讲述了Visual Studio中Git插件的使用，包括提交代码、拉取代码、回滚代码、分支的创建、分支的合并（merge）与衍合（rebase）

[在VS中使用GitLab](/anthologies/tools/git/在VS中使用GitLab) 本文共575字，阅读大约需要3分钟

> 因实验室使用的是基于GitLab搭建的私有Git服务器，所以本文讲述了如何使用Visual Studio的Git插件配置GitLab私服

[在IDEA中使用Git](/anthologies/tools/git/在IDEA中使用Git) 本文共921字，阅读大约需要4分钟

> 本文讲述了在IDEA中有关Git的使用

[GitLab服务器配置](/anthologies/tools/git/GitLab服务器配置) 本文共1143字，阅读大约需要5分钟

> 本文以一次我给实验室配置GitLab服务器的经历为主线，讲述了如何使用docker快速搭建起一个GitLab服务器

[GitLab网页的基本操作](/anthologies/tools/git/GitLab网页的基本操作) 本文共837字，阅读大约需要4分钟

> 本文讲述了私服中GitLab网页的相关使用，主要涉及项目的权限设置、项目移交、项目成员管理以及跨组分享

[配置Git的SSH登陆](/anthologies/tools/git/git-ssh) 本文共305字，阅读大约需要2分钟

> 本文讲述了如何生成`ssh` RSA公私钥，以及如何将其运用到git中。

#### 0x02 MySQL及其优化

##### 0x00 MySQL性能剖析及基准测试

[1.1 剖析查询性能](/anthologies/tools/mysql-optimization/0x00%20MySQL性能剖析及基准测试/剖析查询性能) 本文共5645字，阅读大约需要20分钟

> 本文叙述了使用SHOW PROFILE及performance_schema进行性能查询的方法  

[1.2 基准测试](/anthologies/tools/mysql-optimization/0x00%20MySQL性能剖析及基准测试/基准测试) 本文共5339字，阅读大约需要20分钟

> 本文叙述了集成式（full-stack）及单组件式（single-component）基准测试的目的及方法以及测试指标内容，并给出了mysqlslap工具的用法及收集MySQL状态性能数据的脚本

##### 0x01 数据库结构数据类型优化

[2.1 数据类型优化](/anthologies/tools/mysql-optimization/0x01%20数据库结构数据类型优化/数据类型优化) 本文共3189字，阅读大约需要12分钟

> 本文叙述了在进行数据类型优化和选择时的基本原则、MySQL各种数据类型的时间及空间效率及适用场景、标识列的选择原则