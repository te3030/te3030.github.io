# Docker log


## 限制docker log

### 全局限制

修改`/etc/docker/daemon.json`文件，添加如下内容：
```json
{
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "50m",
        "max-file": "1"
    }
}
```

参数说明：
- log-opts max-size 容器日志文件上限大小
- log-opts max-file 窗口日志文件上限个数


重启docker容器
```bash
systemctl daemon-reload
systemctl restart docker
```
### 单个容器限制

```bash
docker run --log-opt max-size=10m --log-opt max-file=3
```

### docker-compose限制

```yaml
nginx: 
  image: nginx:1.12.1
  restart: always 
  logging: 
    driver: "json-file"
    options: 
      max-size: "50m"
　　  max-file: "2"
```


## 删除现有 docker log

```bash
# 找到日志文件目录
cd /var/lib/docker/containers/*/json.log
# 清空文件内容
cat /dev/null > json.log
```


