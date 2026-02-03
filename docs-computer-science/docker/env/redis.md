# Redis快速安装
redis快速安装

<!--more-->

## 搭建单节点

```shell
docker run -itd --name my-redis -p 6380:6379 redis-test --requirepass 123456
# 为现有的redis创建密码或修改密码的方法：

# 1.进入redis的容器 docker exec -it 容器ID bash
# 2.进入redis目录 cd /usr/local/bin 
# 3.运行命令：redis-cli
# 4.查看现有的redis密码：config get requirepass
# 5.设置redis密码config set requirepass ****（****为你要设置的密码）
# 6.若出现(error) NOAUTH Authentication required.错误，则使用 auth 密码 来认证密码

# 持久性存储
docker run --name some-redis -itd -v ./redis_data:/data redis redis-server --save 60 1 --loglevel warning
#如果至少执行了1次写入操作，则每60秒将保存一次数据库快照（这也将导致更多日志，因此该选项可能是可取的）。
#如果启用了持久性，则数据存储在 中，可以与 或 一起使用（请参阅 docs.docker 卷）。

```

参考文件
> https://hub.docker.com/_/REDIS/
