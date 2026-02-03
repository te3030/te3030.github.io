# Mariadb快速安装
mariadb快速安装

<!--more-->

## 搭建单节点

```shell

docker run -itd --name my-mariadb --env MARIADB_USER=example-user --env MARIADB_PASSWORD=my_cool_secret --env MARIADB_ROOT_PASSWORD=my-secret-pw  mariadb:latest

```


## 搭建mariadb主从

1. 在所有机器执行以下名称
```shell
docker run --name mydb -itd --net=host -e MYSQL_ROOT_PASSWORD=root -v /mydb/conf/:/etc/mysql/conf.d/ mariadb:10.4.13
```

2. 在master节点上，添加或更改一下内容
```conf
[mysqld]
server_id=1
log_bin=mysql-bin
```

3. 在master节点上，执行以下命令
```sql
--- 授权同步二进制文件的用户
grant replication slave on *.* to 'reliuser'@'10.35.78.%' identified by 'tongbu';

---  #查看最新binlog列表
show master status;
--- # 或者show master logs #查看binlog列表
--- 因为 grant 操作mysql.user表，因此二进制日志也会记录grant，Position 从开始的245增长到400

show binlog events in 'mariadb-bin.000001';

select user,password from mysql.user;

--- ps
--- 二进制日志只会记录写操作（包括删除），查询什么的不会记录
--- 二进制日志只会记录操作成功的，失败的不会记录
```


4. 在slave配置文件增添如下
```conf
[mysqld]
server_id=2
read_only=ON
# (主到从是单向同步，从数据库设置只读，防止数据写入从服务器，主从数据不一致问题，但是mysql的root仍旧可以写入)
```

5. 在slave执行以下命令
```sql
help change master to;
start slave;
show slave status\G
show processlist;
```


## 扩展

```shell
将binlog转换为可行性的sql文件
mysqlbinlog mariadb-bin.000001 >dump.sql
```
