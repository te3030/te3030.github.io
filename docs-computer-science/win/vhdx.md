# VHDX

虚拟磁盘文件

<!--more-->



## vhd文件压缩瘦身到实际大小cmd下diskpart

### 选择虚拟磁盘文件

file 是指定的vhdx文件

`select vdisk file="C:\debian\vm.vhdx"`

### 连接虚拟磁盘文件

`attach vdisk readonly`

### 压缩虚拟磁盘文件
`compact vdisk`

### 卸载虚拟磁盘文件
`detach vdisk`

### 例子

```diskpart
DISKPART> select vdisk file="C:\debian\vm.vhdx"

DiskPart 已成功选择虚拟磁盘文件。

DISKPART> attach vdisk readonly

  100 百分比已完成

DiskPart 已成功连接虚拟磁盘文件。

DISKPART> compact vdisk

  100 百分比已完成

DiskPart 已成功压缩虚拟磁盘文件。

DISKPART> detach vdisk

DiskPart 已成功分离虚拟磁盘文件。

DISKPART>
```


> 参考链接：
>
> [使用 VHDX（本机启动）部署 Windows](https://learn.microsoft.com/zh-cn/windows-hardware/manufacture/desktop/deploy-windows-on-a-vhd--native-boot?view=windows-11)
> [如何使用 Diskpart 压缩动态 VHD , How to Compact a Dynamic VHD with Diskpart](https://learn.microsoft.com/en-us/archive/technet-wiki/8043.how-to-compact-a-dynamic-vhd-with-diskpart)


