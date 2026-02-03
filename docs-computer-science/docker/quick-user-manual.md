# Docker快速使用手册
docker快速使用手册，安装执行等常用命令用法

<!--more-->


## 安装Linux

### Centos7安装
略

### Ubuntu安装

> [参考文档](https://docs.docker.com/engine/install/ubuntu/)

- 卸载旧版本
```shell
sudo apt-get remove docker docker-engine docker.io containerd runc
```

- 安装
```shell
# 更新包索引并安装包以允许通过 HTTPS 使用存储库
sudo apt-get update
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 官方 GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 使用以下命令设置存储库
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 更新包索引，并安装最新版本的 Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 验证是否正确安装了 Docker
sudo service docker start

```

### 脚本一键安装
略

### 使用可执行文件运行
略


## 常用命令

### pull

### run

### ps

### logs

### start

### restart

### stop

### rm

### rmi

### update

1. 更新启动策略（--restart）

- **no** 默认策略，在容器退出时不重启容器
- **on-failure** 在容器非正常退出时（退出状态非0），才会重启容器
- **on-failure:n** 在容器非正常退出时重启容器，最多重启n次
- **always** 在容器退出时总是重启容器

例 `docker update --restart=always ubuntu`

### inspect

### exec

### cp

### tag

### build

### login

### push



## 常见问题处理

### docker容器输出日志太大

默认不限制docker容器输出的日志大小，在不限制的清空下，会将磁盘写满
日志文件默认在`/var/lib/docker/containers/<container_id>/<container_id>-json.log`存放；

#### 方式一
**删除文件并重启容器**
```shell
rm -rf /var/lib/docker/containers/<container_id>/<container_id>-json.log
docker restart <container_id>

# container_id 为容器的完整containerId值
```

### 方式二
**清空log文件的数据**
```shell
cat /dev/null > /var/lib/docker/containers/<container_id>/<container_id>-json.log
# container_id 为容器的完整containerId值
# 无需重启容器
```

### 
**在运行时添加log文件限制**
```shell
docker run -itd --log-opt max-size=1g <name>

# 例
# docker run -d --log-opt max-size=1g nginx
```
**docker-compose限制**
添加`max-size`限制
```yaml
  logging:
    driver: "json-file"
    options:
      max-size: "5g"
```

**修改默认容器日志的限制**
编辑文件`/etc/docker/daemon.json`添加如内容
`max-size`表示限制大小
`max-file`表示一个容器有三个日志，分别是`<id>-json.log`、`<id>-json.log.1`、`<id>-json.log.2`
```json
{
  "log-driver":"json-file",
  "log-opts": {"max-size":"500m", "max-file":"3"}
}
```

