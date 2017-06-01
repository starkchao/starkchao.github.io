---
title: xdebug -- debug tool for php
date: 2017-06-01 16:39:09
tags: 
- xdebug 
- php 
- phpstorm
---
如果你现在的调试还是在使用var_dump()或者print_r()，那么xdebug可能会让你的调试更加得心应手，就像平常断点单步调试一样，你可以直观的看见所有的过程与值。
<!--more-->
与任何东西的学习方式一样，**官方文档** 永远是最好的老师：
> [xdebug官方文档](https://xdebug.org/docs/) 
  :) 什么？你不想看英文文档？

***
还好你找到了这里~下边是linux下xdebug的安装及配置方法：

#### Step 1 下载并解压xdebug扩展包
```
wget https://xdebug.org/files/xdebug-2.5.4.tgz
tar -zxvf xdebug-2.5.4.tgz

```
Tips：
- 具体下载哪个版本的包，可以到xdebug的download页面查找：[download](https://xdebug.org/download.php)
- windows用户下载.dll后缀的文件，linux用户下载source包

#### Step 2 编译xdebug
```
cd debug-2.5.4
phpize（编译你要添加的扩展模块的命令）
./configure
make
cp modules/xdebug.so /usr/local/php-7.0.9/lib/php/extensions/no-debug-non-zts-20151012

```
#### Step 3 配置php.ini

对于**PHP5.3之前**的版本：
> add: **zend_extension_ts**="/wherever/you/put/it/xdebug.so"

对于**PHP5.3之后**的版本（我的php版本是7.0.9）：
> From PHP 5.3 onwards, **you always need to use the zend_extension PHP.ini setting name**, and not zend_extension_ts, nor zend_extension_debug.

在我的php.ini中，关于xdebug的配置如下：
![xdebug-config](/img/2017/06/01/xdebug-config.jpg)
[//]: <> (<img src="/img/2017/06/01/xdebug-config.jpg" width="100" height="100" alt="xdebug-config"/>)

其中remote_host与remote_port按照自己的需求指定。
保存php.ini后重启apache或nginx。

#### Step 4 配置phpstorm的xdebug
##### 1. 指定xdebug port
![xdebug-port](/img/2017/06/01/xdebug-port.gif)
这里的port需要与php.ini配置的remote_port一致。
##### 2. 配置DBGp Proxy
![DBGp-proxy](/img/2017/06/01/DBGp-proxy.gif)
IDE Key：与php.ini中xdebug.idekey相同
Host：项目所在位置
Port：项目监听端口

#### Step 5 安装chrome扩展 Xdebug helper
![xdebug-helper](/img/2017/06/01/xdebug-helper.png)
在IDE Key中选择PhpStorm，并设置Key为PHPSTORM（与之前的xdebug.idekey相同即可）
***
到此为止，所有的准备工作就已经完成了，下面来测试一下~！
#### Step 6 测试
1. 在phpstorm中设置代码断点，然后开启debug监听
![start-listening](/img/2017/06/01/start-listening.png)
2. 打开浏览器中Xdebug helper的debug模式
![debug](/img/2017/06/01/debug.png)
3. 访问存在断点的代码位置
如果没问题的话，在phpstorm中应该可以看到：
![accept](/img/2017/06/01/accept.png)
4. 进入debug控制台
点击accept就可以看到debug控制台了
![debug-console](/img/2017/06/01/debug-console.png)

that's done~!