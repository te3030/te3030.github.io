# Docker load



## 修改docker load 的镜像名称为指定的名称

```bash
DOCKER_IMAGES_NAME="jre-alpine:17.0.10"

docker rm -f nps-agnet || echo "docker rm nps-agent error"

a=$(docker load < x86_jre_alpine_17.0.10.tar)

DOCKER_IMAGES_NAME_BAK=$(echo $a | awk '{print $NF}')
if [ "${DOCKER_IMAGES_NAME}" != "${DOCKER_IMAGES_NAME_BAK}" ];then
    docker tag ${DOCKER_IMAGES_NAME_BAK} ${DOCKER_IMAGES_NAME}
fi

```
