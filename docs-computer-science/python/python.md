# pip 换源
1. 临时使用

```shell
# 使用 -i 参数
pip install xxxx -i https://pypi.tuna.tsinghua.edu.cn/simple
```


2. 修改用户目录配置

- Linux : 修改 ~/.pip/pip.conf 文件
- Windows : 修改 %HOMEPATH%\pip\pip.ini 文件

> 如果目录如果不存在，自行创建

```ini
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host = https://pypi.tuna.tsinghua.edu.cn
```

3. 命令行修改配置

```shell
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

**其他国内镜像**
```
阿里云 http://mirrors.aliyun.com/pypi/simple/
清华大学 https://pypi.tuna.tsinghua.edu.cn/simple/
中国科学技术大学 http://pypi.mirrors.ustc.edu.cn/simple/
华中科技大学 http://pypi.hustunique.com/
```

