# docker-basis
## 安装

```shell
# 卸载旧版本
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-selinux \
                  docker-engine-selinux \
                  docker-engine

# 安装依赖包
yum install -y yum-utils

# 安装 Docker CE
# 更新 yum 软件源缓存，并安装 docker-ce
yum makecache fast
yum install docker-ce docker-ce-cli containerd.io

# 使用阿里云的源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 阿里云加速器
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
	"registry-mirrors": ["https://2qxzgh33.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

# 启动 Docker CE
sudo systemctl enable docker
sudo systemctl start docker

# 建立 docker 用户组
# 默认情况下，docker 命令会使用 Unix socket 与 Docker 引擎通讯。
# 而只有 root 用户和 docker 组的用户才可以访问 Docker 引擎的 Unix socket。
# 出于安全考虑，一般 Linux 系统上不会直接使用 root 用户。
# 因此，更好地做法是将需要使用 docker 的用户加入 docker 用户组

# 建立 docker 组
sudo groupadd docker

# 将当前用户加入 docker 组
sudo usermod -aG docker $USER

```

## 常用命令

```shell
# 获取镜像
docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]
比如：

$ docker pull centos:7
7: Pulling from library/centos
75f829a71a1c: Pull complete 
Digest: sha256:19a79828ca2e505eaee0ff38c2f3fd9901f4826737295157cc5212b7a372cd2b
Status: Downloaded newer image for centos:7
docker.io/library/centos:7

# 查看镜像
docker images

# 运行
docker run -it centos /bin/bash

# 删除镜像
docker rmi -f 镜像ID

# 查看容器
docker ps 查看正在运行的容器
				-a 查看运行过的容器

# 删除容器
docker rm -f 容器ID

# 重新进入容器
docker exec -it 容器ID /bin/bash  进入容器打开新的bash
docker attach 容器ID              进入容器退出时状态

# 将文件从docker拷贝到主机
docker cp 容器ID:/home/test.cpp /home/

# 查看log
docker logs -f -t --tail 10 容器ID
```

## 练习

### 部署nginx

```shell
docker pull nginx
docker run -d --name nginx01 -p 3344:80 nginx
# -d 后台运行
# --name 给容器命名
# -p 宿主机端口，容器内部端口
curl localhost:3344
docker ps
```

端口暴露的概念：

![image](https://github.com/ChrysosKun/docker-basis/blob/master/images/1.png)

## 可视化

```shell
docker run -d -p 8088:9000 --restart=always -v /var/run/docker.sock:/var/run/docker.sock --privileged=true portainer/portainer
```

![image](https://github.com/ChrysosKun/docker-basis/blob/master/images/2.png)

![image](https://github.com/ChrysosKun/docker-basis/blob/master/images/3.png)

## 容器数据卷

### 什么是容器数据卷

#### docker的理念回顾

*将应用和环境打包成一个镜像*

- 如果数据都在容器中，容器删除，数据就会丢失。需求：数据可以持久化
- MySQL,容器删除=删库跑路。需求：MySQL数据可以存储到本地
- 容器之间可以有一个共享技术。Docker容器产生的数据可以同步到本地
- 卷技术：目录的挂载，将容器目录挂载到Linux

![image](https://github.com/ChrysosKun/docker-basis/blob/master/images/4.png)

#### 使用数据卷

方式一：直接使用命令行来挂载 -v

```shell
docker run -it -v 主机目录:容器内目录

#测试
docker run -it -v /home/seshi:/home centos /bin/bash

#启动起来使用docker inspect 容器ID
```

停止容器数据依然是同步的

#### 实战：安装MySQL

```shell
# 获取镜像
docker pull mysql:5.7

# 运行容器，需要做数据挂载
# 安装启动MySQL需要配置密码
# 官方测试：docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag

# 参数
-d 后台运行
-p 端口映射
-v 卷挂载
-e 环境配置
--name 容器名字
awesome
# 启动
docker run -d -p 3310:3306 --restart=always -v /home/mysql/conf:/etc/mysql/conf.d -v /home/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mymysql --name mysql01 mysql:5.7
```

#### 具名挂载和匿名挂载

```shell
# 匿名挂载
-v 容器路径
docker run -d -P --name nginx01 -V /etc/nginx nginx

# 查看所有的volume的情况
docker volume ls

[root@centos-ecs ~]# docker volume ls
DRIVER              VOLUME NAME
local               39fe6e4095e817bf12c2c7fb8d5d3e752e2b5f4f74b76e8e5699a49b5797525a

# 这种就是匿名挂载，在 -v 只写了容器内的路径，没有写容器外的路径

# 具名挂载

docker run -d -P --name nginx02 -v juming-nginx:/etc/nginx nginx

[root@centos-ecs ~]# docker volume ls
DRIVER              VOLUME NAME
local               39fe6e4095e817bf12c2c7fb8d5d3e752e2b5f4f74b76e8e5699a49b5797525a
local               juming-nginx

# -v 卷名：容器内路径

# 查看这个卷
[root@centos-ecs ~]# docker volume inspect juming-nginx
[
    {
        "CreatedAt": "2020-09-07T11:00:26+08:00",
        "Driver": "local",
        "Labels": null,
        "Mountpoint": "/var/lib/docker/volumes/juming-nginx/_data",
        "Name": "juming-nginx",
        "Options": null,
        "Scope": "local"
    }
]
```

所有的docker容器内的卷，没有指定目录的情况下都在 /var/lib/docker/volumes/xxxx/_data

通过具名挂载可以方便的找到一个卷

```
# 如何确定是具名挂载还是匿名挂载，还是指定路径挂载
-v 容器内路径                 # 匿名挂载
-v 卷名:容器内路径             # 具名挂载
-v /宿主机路径:容器内路径       # 指定路径挂载
```

## 初识Dockerfile

方式二：创建镜像时挂载

Dockerfile就是用来构建docker镜像的构建文件，命令脚本。

通过脚本可以生成镜像，镜像是一层一层的，脚本一个个的命令，每个命令都是一层。

```shell
# 创建一个dockerfile文件，名字可以随机，建议Dockerfile
# 文件中的内容 指令（大写）参数

FROM centos

VOLUME ["volume01","volume02"]      # 此为匿名挂载

CMD echo "----------end----------"

CMD /bin/bash

# 这里每个命令就像镜像的一层

# 构建
docker build -f dockerfile01 -t langyaya/centos .

[root@centos-ecs docker-volume]# docker build -f dockerfile01 -t langyaya/centos .
Sending build context to Docker daemon  2.048kB
Step 1/4 : FROM centos
 ---> 0d120b6ccaa8
Step 2/4 : VOLUME ["volume01","volume02"]
 ---> Running in e501cffb694a
Removing intermediate container e501cffb694a
 ---> f336ad7bfb33
Step 3/4 : CMD echo "----------end----------"
 ---> Running in e903ab3d8fa2
Removing intermediate container e903ab3d8fa2
 ---> 3a820a23a5f7
Step 4/4 : CMD /bin/bash
 ---> Running in 9eeb562caf95
Removing intermediate container 9eeb562caf95
 ---> 2d5ac7cc876b
Successfully built 2d5ac7cc876b
Successfully tagged langyaya/centos:latest

[root@centos-ecs docker-volume]# docker images
REPOSITORY                                                 TAG                 IMAGE ID            CREATED             SIZE
langyaya/centos                                            latest              2d5ac7cc876b        34 seconds ago      215MB

# 构建完成

# 用自己构建的镜像创建容器
docker run -it 2d5ac7cc876b

[root@22e5a20e1e9c /]# ls -l
total 56
lrwxrwxrwx  1 root root    7 May 11  2019 bin -> usr/bin
drwxr-xr-x  5 root root  360 Sep  7 06:43 dev
drwxr-xr-x  1 root root 4096 Sep  7 06:43 etc
drwxr-xr-x  2 root root 4096 May 11  2019 home
lrwxrwxrwx  1 root root    7 May 11  2019 lib -> usr/lib
lrwxrwxrwx  1 root root    9 May 11  2019 lib64 -> usr/lib64
drwx------  2 root root 4096 Aug  9 21:40 lost+found
drwxr-xr-x  2 root root 4096 May 11  2019 media
drwxr-xr-x  2 root root 4096 May 11  2019 mnt
drwxr-xr-x  2 root root 4096 May 11  2019 opt
dr-xr-xr-x 89 root root    0 Sep  7 06:43 proc
dr-xr-x---  2 root root 4096 Aug  9 21:40 root
drwxr-xr-x 11 root root 4096 Aug  9 21:40 run
lrwxrwxrwx  1 root root    8 May 11  2019 sbin -> usr/sbin
drwxr-xr-x  2 root root 4096 May 11  2019 srv
dr-xr-xr-x 13 root root    0 Sep  7 06:43 sys
drwxrwxrwt  7 root root 4096 Aug  9 21:40 tmp
drwxr-xr-x 12 root root 4096 Aug  9 21:40 usr
drwxr-xr-x 20 root root 4096 Aug  9 21:40 var
drwxr-xr-x  2 root root 4096 Sep  7 06:43 volume01
drwxr-xr-x  2 root root 4096 Sep  7 06:43 volume02

# 发现卷已经挂载

```

其它见：[ChrysosKun的小岛](https://sh.chrysoskun.one/)
