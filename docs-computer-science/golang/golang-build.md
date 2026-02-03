# golang打包
golang打包以及相关参数调整

<!--more-->



## golang语言打包添加ico图标

> 使用的是[rsrc](https://github.com/akavel/rsrc)库实现

### 安装 **rsrc**
```shell
go install github.com/akavel/rsrc@latest
```

### 创建**main.manifest**文件

> 在main.go的同级目录下创建main.manifest
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
    <assemblyIdentity version="1.0.0.0" processorArchitecture="*" name="SomeFunkyNameHere" type="win32"/>
    <dependency>
        <dependentAssembly>
            <assemblyIdentity type="win32" name="Microsoft.Windows.Common-Controls" version="6.0.0.0" processorArchitecture="*" publicKeyToken="6595b64144ccf1df" language="*"/>
        </dependentAssembly>
    </dependency>
    <application xmlns="urn:schemas-microsoft-com:asm.v3">
        <windowsSettings>
            <dpiAwareness xmlns="http://schemas.microsoft.com/SMI/2016/WindowsSettings">PerMonitorV2, PerMonitor</dpiAwareness>
            <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">True</dpiAware>
        </windowsSettings>
    </application>
</assembly>
```


### 生成**main.syso**文件

```shell
rsrc -manifest main.manifest -ico main.ico -o main.syso
```


### build

```shell
go build -ldflags="-H windowsgui  -w -s"
# -H 不显示CMD窗口
```

