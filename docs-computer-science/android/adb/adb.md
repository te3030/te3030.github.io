# Android ADB 命令

## 获取手机的ip

```bash
adb shell ip addr show wlan0 | grep "inet " | awk '{print $2}' | cut -d/ -f1
```


## 启动应用

```bash
db shell am start com.android.settings/.Settings
```


## 清空logcat
```bash
adb logcat -c
```


## 设备上运行的activity
```bash
```

## 使用浏览器打开指定网址

```bash
adb shell am start -a android.intent.action.VIEW -d http://www.baidu.com
```


## 杀死应用进程

```bash
adb shell am force-stop com.android.settings
```

## 终止所有后台进程

```bash
adb shell am kill-all com.android.settings
```

## 删除应用所有数据

```bash
adb shell pm clear com.android.settings
```

## 获取设备已安装应用列表

```bash
adb shell pm list packages
# -f：查看关联文件。
# -d：进行过滤以仅显示已停用的软件包。
# -e：进行过滤以仅显示已启用的软件包。
# -s：进行过滤以仅显示系统软件包。
# -3：进行过滤以仅显示第三方软件包。
# -i：查看软件包的安装程序。
# -u：包括已卸载的软件包。
# –user user_id：要查询的用户空间。
```




