### 配置
---
这里使用 Windows 端安装靶机。
先去 [Vagrant 官网](https://developer.hashicorp.com/vagrant/install) 安装 Vagrant 工具，下载 **Windows - Binary download - AMD64**
安装后重启，**以管理员身份运行 CMD**，进入目标文件夹后输入：
```Powershell
mkdir metasploitable3-workspace

cd metasploitable3-workspace

Invoke-WebRequest -Uri "https://raw.githubusercontent.com/rapid7/metasploitable3/master/Vagrantfile" -OutFile "Vagrantfile"

vagrant up
```
在配置的过程中可能会出现以下问题：
#### 问题1. `No space left on device`
---
```
The box failed to unpackage properly. Please verify that the box  
file you're trying to add is not corrupted and that enough disk space  
is available and then try again.  
The output from attempting to unpackage (if any):

x Vagrantfile  
x box.ovf  
x metadata.json  
x metasploitable3-win2k8-disk001.vmdk: Write failed: No space left on device  
bsdtar.EXE: Error exit delayed from previous errors.
```
这种情况是因为**Vagrant 默认把下载的基础镜像（Box）和缓存文件存放在 C 盘的用户目录下**，但是 **C 盘没有足够空间**。解决方案如下：
#### 第一步：清理残余文件并更改存储路径
1. 进入`C:\Users\用户名\`，删除`.vagrant.d`文件夹。
2. 在 D 盘新建文件夹（如`VagrantHome`）。
3. 打开**编辑系统环境变量**的**高级**菜单，点击右下角的 **“环境变量”**，然后在上面的 **“用户变量”** 区域，点击 **“新建”**，输入**变量名**`VAGRANT_HOME`、**变量值**`D:\VagrantHome`，一路点击“确定”保存。
4. 重启 **PowerShell 窗口**，让新设置生效。
#### 第二步：更改 VirtualBox 的虚拟磁盘存放路径
1. 打开 **Oracle VM VirtualBox 管理器**。
2. 点击左上角的 **“管理” (File)** -> **“首选项” (Preferences)**（快捷键 `Ctrl+G`）。
3. 在 **“常规” (General)** 选项卡里，找到 **“默认虚拟电脑位置” (Default Machine Folder)**。
4. 把它改成 D 盘的一个目录，比如 `D:\VirtualBoxVMs`。
5. 点击确定。
之后 CMD 重新输入`vagrant up`测试。
#### 问题2. `E_ACCESSDENIED (0x80070005)`
---
```
There was an error while executing VBoxManage, a CLI used by Vagrant  
for controlling VirtualBox. The command and stderr is shown below.

Command: ["showvminfo", "7a955596-221a-4c48-ac43-d70ff793dd1b", "--machinereadable"]

Stderr: VBoxManage.exe: error: The object functionality is limited  
VBoxManage.exe: error: Details: code E_ACCESSDENIED (0x80070005), component MachineWrap, interface IMachine, callee IUnknown  
VBoxManage.exe: error: Context: "LockMachine(a->session, LockType_Shared)" at line 3327 of file VBoxManageInfo.cpp
```
Vagrant（以管理员身份运行）试图控制 VirtualBox，但 VirtualBox 的后台进程（VBoxSVC.exe）可能正被另一个权限较低的进程（比如之前手动打开的 VirtualBox 界面）占用着，导致 Vagrant **抢不到控制权（Access Denied）**，于是报错崩溃。解决方案如下：
1. 打开 VirtualBox 图形界面。
2. 看左边列表里有没有 **Metasploitable3-ub1404**。
    - **如果有**：右键点击它 -> **删除 (Remove)**。可以去虚拟机储存的文件夹下确保完全删除。
    - **如果列表是空的**：那更好，直接关掉 VirtualBox 界面。
3. **务必关闭 VirtualBox 图形界面**（不要留着它，否则又会抢占进程）。可以用
```Powershell
taskkill /F /IM VBox*
```
来检查后台进程是否开着。如果出现
```
错误: 没有找到进程 "VBox*"。
```
说明 VirtualBox 已完全关闭。
之后 CMD 重新输入`vagrant up`测试。
#### 问题3. 网络冲突 (DHCP Conflict)
---
```
A host only network interface you're attempting to configure via DHCP  
already has a conflicting host only adapter with DHCP enabled. The  
DHCP on this adapter is incompatible with the DHCP settings. Two  
host only network interfaces are not allowed to overlap, and each  
host only network interface can have only one DHCP server. Please  
reconfigure your host only network or remove the virtual machine  
using the other host only network.
```
Ubuntu 刚刚启动时占用（或创建）了一个网络适配器，并设置了一套 IP 规则。紧接着 Windows 试图启动，它想要用一套不一样的 IP 规则（或者试图创建新的），结果发现现有的规则冲突了，于是 VirtualBox 为了防止网络错乱，直接叫停了。解决方案如下：
1. CMD 输入
```Powershell
vagrant halt
```
把刚刚启动成功的 Ubuntu 先关机。
2. 打开 **VirtualBox 图形界面**。
3. 点击左上角的 **“管理” (File)** -> **“工具” (Tools)** -> **“网络” (Network)**。
    - 注意：不同版本的 VirtualBox 这个菜单位置可能略有不同，有的叫“主机网络管理器 (Host Network Manager)”。
4. 在弹出的列表中，若有一个或多个类似 `VirtualBox Host-Only Ethernet Adapter` 的东西，必须将其**全部删除**。如果有提示“无法删除”，请确保第一步的 `vagrant halt` 已经执行完毕，或者强制结束 VirtualBox 后台进程。
5. 删除干净后，点击“关闭”或“应用”。
之后 CMD 重新输入`vagrant up`测试。

如果输入`vagrant up`后出现
```
Bringing machine 'ub1404' up with 'virtualbox' provider...
Bringing machine 'win2k8' up with 'virtualbox' provider...
==> ub1404: Checking if box 'rapid7/metasploitable3-ub1404' version '0.1.12-weekly' is up to date...
==> ub1404: Clearing any previously set forwarded ports...
==> ub1404: Clearing any previously set network interfaces...
==> ub1404: Preparing network interfaces based on configuration...
    ub1404: Adapter 1: nat
    ub1404: Adapter 2: hostonly
==> ub1404: Forwarding ports...
    ub1404: 22 (guest) => 2222 (host) (adapter 1)
==> ub1404: Running 'pre-boot' VM customizations...
==> ub1404: Booting VM...
==> ub1404: Waiting for machine to boot. This may take a few minutes...
    ub1404: SSH address: 127.0.0.1:2222
    ub1404: SSH username: vagrant
    ub1404: SSH auth method: password
==> ub1404: Machine booted and ready!
==> ub1404: Checking for guest additions in VM...
    ub1404: No guest additions were detected on the base box for this VM! Guest
    ub1404: additions are required for forwarded ports, shared folders, host only
    ub1404: networking, and more. If SSH fails on this machine, please install
    ub1404: the guest additions and repackage the box to continue.
    ub1404:
    ub1404: This is not an error message; everything may continue to work properly,
    ub1404: in which case you may ignore this message.
==> ub1404: Setting hostname...
==> ub1404: Configuring and enabling network interfaces...
==> ub1404: Machine already provisioned. Run `vagrant provision` or use the `--provision`
==> ub1404: flag to force provisioning. Provisioners marked to run always will still run.
==> win2k8: Checking if box 'rapid7/metasploitable3-win2k8' version '0.1.0-weekly' is up to date...
==> win2k8: Fixed port collision for 22 => 2222. Now on port 2200.
==> win2k8: Clearing any previously set network interfaces...
==> win2k8: Preparing network interfaces based on configuration...
    win2k8: Adapter 1: nat
    win2k8: Adapter 2: hostonly
==> win2k8: Forwarding ports...
    win2k8: 3389 (guest) => 3389 (host) (adapter 1)
    win2k8: 22 (guest) => 2200 (host) (adapter 1)
    win2k8: 5985 (guest) => 55985 (host) (adapter 1)
    win2k8: 5986 (guest) => 55986 (host) (adapter 1)
==> win2k8: Running 'pre-boot' VM customizations...
==> win2k8: Booting VM...
==> win2k8: Waiting for machine to boot. This may take a few minutes...
    win2k8: WinRM address: 127.0.0.1:55985
    win2k8: WinRM username: vagrant
    win2k8: WinRM execution_time_limit: PT2H
    win2k8: WinRM transport: negotiate
==> win2k8: Machine booted and ready!
==> win2k8: Checking for guest additions in VM...
    win2k8: The guest additions on this VM do not match the installed version of
    win2k8: VirtualBox! In most cases this is fine, but in rare cases it can
    win2k8: prevent things such as shared folders from working properly. If you see
    win2k8: shared folder errors, please make sure the guest additions within the
    win2k8: virtual machine match the version of VirtualBox you have installed on
    win2k8: your host and reload your VM.
    win2k8:
    win2k8: Guest Additions Version: 6.0.8
    win2k8: VirtualBox Version: 7.2
==> win2k8: Setting hostname...
==> win2k8: Waiting for machine to reboot...
==> win2k8: Configuring and enabling network interfaces...
==> win2k8: Running provisioner: shell...
    win2k8: Running: inline PowerShell script
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall set allprofiles state on
    win2k8: Ok.
    win2k8:
==> win2k8: Running provisioner: shell...
    win2k8: Running: inline PowerShell script
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 8484 for Jenkins" dir=in action=allow protocol=TCP localport=8484
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 8282 for Apache Struts" dir=in action=allow protocol=TCP localport=8282
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 80 for IIS" dir=in action=allow protocol=TCP localport=80
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 4848 for GlassFish" dir=in action=allow protocol=TCP localport=4848
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 8080 for GlassFish" dir=in action=allow protocol=TCP localport=8080
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 8585 for Wordpress and phpMyAdmin" dir=in action=allow protocol=TCP localport=8585
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Java 1.6 java.exe" dir=in action=allow program="C:\openjdk6\openjdk-1.6.0-unofficial-b27-windows-amd64\jre\bin\java.exe" enable=yes
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 3000 for Rails Server" dir=in action=allow protocol=TCP localport=3000
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 8020 for ManageEngine Desktop Central" dir=in action=allow protocol=TCP localport=8020
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 8383 for ManageEngine Desktop Central" dir=in action=allow protocol=TCP localport=8383
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 8022 for ManageEngine Desktop Central" dir=in action=allow protocol=TCP localport=8022
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 9200 for ElasticSearch" dir=in action=allow protocol=TCP localport=9200
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Open Port 161 for SNMP" dir=in action=allow protocol=UDP localport=161
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Closed port 445 for SMB" dir=in action=block protocol=TCP localport=445
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Closed port 139 for NetBIOS" dir=in action=block protocol=TCP localport=139
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Closed port 135 for NetBIOS" dir=in action=block protocol=TCP localport=135
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Closed Port 3389 for Remote Desktop" dir=in action=block protocol=TCP localport=3389
    win2k8: Ok.
    win2k8:
    win2k8:
    win2k8: C:\Windows\system32>netsh advfirewall firewall add rule name="Closed Port 3306 for MySQL" dir=in action=block protocol=TCP localport=3306
    win2k8: Ok.
    win2k8:
==> win2k8: Running provisioner: shell...
    win2k8: Running: inline PowerShell script
    win2k8:
    win2k8: C:\Windows\system32>copy C:\vagrant\scripts\installs\setup_linux_share.bat C:\Windows
    win2k8: The system cannot find the path specified.
    win2k8:
    win2k8: C:\Windows\system32>reg add HKLM\Software\Microsoft\Windows\CurrentVersion\Run /v share /t REG_SZ /d "C:\Windows\setup_linux_share.bat" /f
    win2k8: The operation completed successfully.
==> win2k8: Running provisioner: shell...
    win2k8: Running: inline PowerShell script
    win2k8:
    win2k8: CMDKEY: Credential added successfully.
    win2k8: The command completed successfully.
    win2k8:
==> win2k8: Running provisioner: shell...
    win2k8: Running: inline PowerShell script
PS D:\PrivEsc VM\metasploitable3-workspace>
```
说明 **Metasploitable 3** 靶机已经配置完毕！
和靶机相关的 CMD 命令如下：
启动：`vagrant up`
关机：`vagrant halt`

## END