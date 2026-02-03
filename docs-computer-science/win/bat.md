# Windows Bat命令
Bat命令

<!--more-->


## %cd%

驱动器盘符:+当前目录

```cmd
> echo %cd%
c:/dir  
```

## %~dp0

用在批处理文件中，它是由它所在的批处理文件的目录位置决定的，是批处理文件所在的盘符:+路径。在执行这个批处理文件的过程中，它展开后的内容是不可以改变的。

```bat
# dirshow.bat文件内容
> @echo off   
> echo this is %%cd%%  %cd%   
> echo this is %%~dp0 %~dp0  

C:/>D:/dirshow.bat
this is %cd%  C:/   
this is %~dp0 D:/  
```


## 让bat批处理以管理员权限运行的实现方法

```bat
@echo off
%1 mshta vbscript:CreateObject("Shell.Application").ShellExecute("cmd.exe","/c %~s0 ::","","runas",1)(window.close)&&exit
cd /d "%~dp0"
```


## 设置网络和Internet 代理服务器

```bat
rem 设置代理服务器地址
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyServer /d "192.168.60.123:10809" /f

rem 请勿对以下列条目开头的地址使用代理服务器
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyOverride /t REG_SZ /d "11.*;68.*;10.*;" /f

rem 勾选"请勿将代理服务器用于本地(Intranet)地址" 即结尾此处添加"<local>"
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyOverride /t REG_SZ /d "11.*;68.*;10.*;<local>" /f

rem 开启"win系统手动设置代理"
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyEnable /t REG_DWORD /d 1 /f

rem 关闭"win系统手动设置代理"
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyEnable /t REG_DWORD /d 0 /f

rem 清空代理服务器地址
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyServer /d "" /f

rem 清空"请勿对以下列条目开头的地址使用代理服务器"地址
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyOverride /t REG_SZ /d "" /f

```


## 设置网卡静态IP地址

```bat
netsh interface ip set addr "WLAN" static 192.168.1.108 255.255.255.0 192.168.1.1 1
```

## 设置网卡自动获取IP地址

```bat
netsh interface ip set addr "WLAN" dhcp
```


## 设置网卡自动获取DNS服务器地址

```
netsh interface ip set dns "WLAN" dhcp
```

## 设置网卡DNS服务器地址

设置首选DNS服务器地址
```bat
netsh interface ip set dns name="WLAN" source=static address=223.5.5.5 register=primary validate=no

      name         - 接口的名称或索引。
      source       - 下列值之一:
                     dhcp: 将 DHCP 设置为源，以便为特定接口配置 DNS
                            服务器。
                     static: 将用于配置 DNS 服务器的源设置为
                             本地静态配置。
      address      - 下列值之一:
                     <IP address>: DNS 服务器的 IP 地址。
                     none: 清除 DNS 服务器的列表。
      register     - 下列值之一:
                     none: 禁用动态 DNS 注册。
                     primary: 仅在主 DNS 后缀下注册。
                     both: 在主 DNS 后缀下注册，同时在特定连接后缀下
                           注册。
      validate     - 指定是否将验证 DNS 服务器
                     设置。默认情况下，值为 yes。
```

设置备用DNS服务器地址
```bat
netsh interface ip add dns name="WLAN" address=114.114.114.114 index=2 validate=no

      name         - 要添加的 DNS 服务器的接口的名称
                     或索引。
      address      - 要添加的 DNS 服务器的 IP 地址。
      index        - 为指定的 DNS 服务器地址指定
                     索引(首选项)。
      validate     - 指定是否将验证 DNS 服务器设置。
                     默认情况下，值为 yes。
```
