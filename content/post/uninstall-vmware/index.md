+++

title = "完全卸载VMware Workstation"
date = 2022-03-19T12:05:03+08:00
slug = "uninstall_vmware"
description = "VMware Workstation如果卸载不彻底，会导致无法再次安装，此时可以手动清理卸载残留"
tags = [ "VMware" ]
categories = [ "Tech" ]
image = ''

+++

之前没有通过引导卸载VMware Workstation，直接把整个文件夹删了，导致卸载不彻底。VMware在安装时会先检测是否已经安装，使得我的电脑无法再次安装VMware Workstation。后来按照官方的卸载教程试了几次都没成功，偶然发现系统环境变量中还有VMware的路径，才解决了这个问题。关键在于卸载后没有删除环境变量，所以被认为没有完全卸载VMware Workstation。  
### 通过Workstation安装程序自动清理
下载对应版本的安装程序，在当前文件夹打开终端，在终端中输入
```powershell
VMware-workstation-full-xxx-xxx.exe /clean
```
### 停止VMware相关的服务
在Windows搜索框搜索``services.msc``，打开“服务”，停止所有VMware相关的服务。
* VMware Authorization Service
* VMware Authentication Service
* VMware Registration Service
* VMware DHCP Service
* VMware NAT Service
* VMware USB Arbitration Service
* VMware Workstation Server
* VMware WSX Service
### 删除VMware network bridge adapter
1. 打开``控制面板\网络和 Internet\网络连接``
2. 右键，属性，选择VMware Bridge Protocol，卸载
### 删除VMware相关的网络适配器
1. 打开``控制面板\硬件和声音\设备管理器``
2. 在“查看”工具栏勾选上“显示隐藏的设备”
3. 点击“网络适配器”，卸载名字包含VMware的适配器
### 删除和VMware有关的文件夹
* 程序安装目录  
* 数据目录  
  默认路径``C:\Program Files(X86)\VMware\``
* 开始菜单中的VMware  
  路径``C:\ProgramData\VMware``
* 快捷方式  
* 其他文件  
  * ``C:\Windows\system32\vmnat.exe``
  * ``C:\Windows\system32\vmnetbridge.exe``
  * ``C:\Windows\system32\VMNetDHCP.exe``
  * ``C:\Windows\system32\vmnetdhcp.leases``
  * ``C:\Windows\system32\vmxw2ksetup.dll``
  * ``C:\Windows\system32\vnetprobe.exe``
  * ``C:\Windows\system32\vnetprobelib.dll``
  * ``C:\Windows\system32\vnetinst.dll``
  * ``C:\Windows\system32\vnetlib.dll``
  * ``C:\Windows\system32\vnetlib.exe``
  * ``C:\Windows\system32\drivers\vmnet.sys``
  * ``C:\Windows\system32\drivers\vmnetx.sys``
  * ``C:\Windows\system32\drivers\VMparport.sys``
  * ``C:\Windows\system32\drivers\vmx86.sys``
  * ``C:\Windows\system32\drivers\vmnetadapter.sys``
  * ``C:\Windows\system32\drivers\vmnetbridge.sys``
  * ``C:\Windows\system32\drivers\vmnetuserif.sys``
  * ``C:\Windows\system32\drivers\hcmon.sys``
  * ``C:\Windows\system32\drivers\vmusb.sys``
* 注册表  
    打开注册表管理器，删除以下注册表
    * ``HKEY_CLASSES_ROOT\Installer\Features\A57F49D06AE015943BFA1B54AFE9506C``
    * ``HKEY_CLASSES_ROOT\Installer\Products\A57F49D06AE015943BFA1B54AFE9506C``
    * ``HKEY_CLASSES_ROOT\Installer\UpgradeCodes\3F935F414A4C79542AD9C8D157A3CC39``
    * ``HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{0D94F75A-0EA6-4951-B3AF-B145FA9E05C6}``
    * ``HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\VMware, Inc.\VMware Workstation``
    * ``HKEY_LOCAL_MACHINE\SOFTWARE\Wow6432Node\VMware, Inc.\Installer\VMware Workstation``
    * ``HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Applications\vmware.exe``
### 删除环境变量
打开系统环境变量设置，删除VMware Workstation的执行路径  
重启电脑，就完成了VMware Workstation的完全卸载。  
各版本路径存在一些不同，具体参考[官方文档](https://kb.vmware.com/s/article/1308)。