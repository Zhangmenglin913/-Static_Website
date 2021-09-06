### docker 安装

要求Centos7 Linux 内核：建议 3.10 以上，3.8以上貌似也可。

```bash
#1
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
#国内
curl -sSL https://get.daocloud.io/docker | sh

#查看内核版本
uname -r

# 卸载旧版本
yum remove docker docker-common docker-selinux docker-engine

# 安装依赖软件包
yum install -y yum-utils device-mapper-persistent-data lvm2

# 设置yum源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum-config-manager -add-repo http://download.docker.com/linux/centos/docker-ce.repo

# available version
yum list docker-ce --showduplicates | sort -r

# install 
yum -y install docker-ce

# start docker
systemctl start docker
systemctl enable docker

# docker version
docker version

```

#### docker使用

```bash
# 启动容器
docker start [contaner-name]

# 进入容器
docker exec -it [container-name] bash

# -i ： 交互
# -t ： 终端

# 停止容器
docker stop <容器 ID>

# 导出容器
docker export 1e560fca3906 > ubuntu.tar #文件名

# 导入容器快照
cat docker/ubuntu.tar | docker import - test/ubuntu:v1

例子： docker import http://example.com/exampleimage.tgz example/imagerepo

# 删除容器
docker rm -f 1e560fca3906

# 本机和容器之间拷贝文件
docker cp [OPTIONS] CONTAINER:SRC_PATH DEST_PATH
docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH

例子： docker cp /www/runoob 96f7f14e99ab:/www/

```

