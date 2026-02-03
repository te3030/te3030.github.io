# Minio快速安装
minio快速安装

<!--more-->


## 搭建minio集群
```shell

docker run -d --name minio --restart=always --net=host \
-e MINIO_ACCESS_KEY=minio \
-e MINIO_SECRET_KEY=minio123 \
-v /root/minio/update:/data1 \
-v /root/minio/bakup:/data2 \
quay.io/minio/minio:RELEASE.2022-04-12T06-55-35Z \
server --address 192.168.153.153:9000 --console-address ":9001" http://192.168.153.15{3...6}/data{1...2}
```

