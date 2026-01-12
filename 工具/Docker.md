# 容器
官方解释：**容器就是将软件打包成标准化单元，以用于开发、交付和部署**
- **容器镜像是轻量的、可执行的独立软件包** ，包含软件运行所需的所有内容：代码、运行时环境、系统工具、系统库和设置
- **容器化软件适用于基于 Linux 和 Windows 的应用，在==任何环境==中都能够始终如一地运行**
- ==**容器赋予了软件独立性**==，使其免受外在环境差异（例如，开发和预演环境的差异）的影响，从而有助于减少团队间在相同基础设施上运行不同软件时的冲突


**容器虚拟化的是==操作系统==而不是硬件，容器之间是共享同一套操作系统资源的**
**虚拟机技术是==虚拟出一套硬件==后，在其上运行一个完整操作系统**
**容器的隔离级别会稍低一些**

---
# Docker
Docker是**世界领先的软件容器平台**
有了Docker，我们就可以把应用程序和它运行时所需要的各种依赖包、第三方文件、配置文件打包在一起，以便在任何环境中都可以正确运行
**Docker 思想**：
- **集装箱**：就像海运中的集装箱一样，Docker 容器包含了应用程序及其所有依赖项，确保在任何环境中都能以相同的方式运行
- **标准化**：运输方式、存储方式、API 接口
- **隔离**：每个 Docker 容器都在自己的隔离环境中运行，与宿主机和其他容器隔离

## Docker基本概念
- 镜像（Image）
- 容器（Container）
- 仓库（Reponsitory）
![[Pasted image 20250503220748.png]]
### Image：特殊的文件系统
操作系统分为内核和用户空间
Docker 镜像（Image），就相当于是一个 root 文件系统，提供容器运行时候所需的程序、库、资源、配置等文件，包含一些为运行时准备的一些配置参数

Docker 设计时，就充分利用 **Union FS** 的技术，将其设计为**分层存储的架构** 。镜像实际是由多层文件系统联合组成

### Container：镜像运行时的实体
Image：Container 的关系就像 类：实例
Image是镜像的定义，Container是镜像运行时的实体
容器的实质是进程，运行于属于自己的独立的命名空间。

### Repository：集中存放镜像文件的地方
镜像构建完成后，可以很容易的在当前宿主上运行，但是， **如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务**


### 关系

![[Pasted image 20250503221526.png]]
- Dockerfile 是一个文本文件，包含了一系列的指令和参数，用于定义如何构建一个 Docker 镜像。运行 `docker build`命令并指定一个 Dockerfile 时，Docker 会读取 Dockerfile 中的指令，逐步构建一个新的镜像，并将其保存在本地。
- `docker pull` 命令可以从指定的 Registry/Hub 下载一个镜像到本地，默认使用 Docker Hub。
- `docker run` 命令可以从本地镜像创建一个新的容器并启动它。如果本地没有镜像，Docker 会先尝试从 Registry/Hub 拉取镜像。
- `docker push` 命令可以将本地的 Docker 镜像上传到指定的 Registry/Hub


---
# Build Ship Run

![[Pasted image 20250503221722.jpg]]
- **Build（构建镜像）**：镜像就像是集装箱包括文件以及运行环境等等资源。
- **Ship（运输镜像）**：主机和仓库间运输，这里的仓库就像是超级码头一样。
- **Run （运行镜像）**：运行的镜像就是一个容器，容器就是运行程序的地方。

==Docker 运行过程也就是去仓库把镜像拉到本地，然后用一条命令把镜像运行起来变成容器==


---
# 容器化
> [!info] 将应用程序打包成容器，然后在容器中运行应用程序的过程
> 1. 创建一个Dockerfile（来告诉Docker构建应用程序镜像所需要的步骤和配置）
> 2. 使用Dockerfile构建镜像
> 3. 使用镜像创建和运行容器

Dockerfile：文本文件，包含了构建 Docker 镜像的所有指令
> [!info] 指令
> 1. FROM：指定基础镜像
> 2. COPY：将文件或目录复制到镜像中（source dest）
> 3. CMD：指定容器创建时的默认命令（CMD ["<可执行文件活命令>","\<param>..."])
> 4. RUN


---
# Docker常见命令
## 基本命令
```shell
docker version # 查看docker版本
docker images # 查看所有已下载镜像，等价于：docker image ls 命令
docker container ls # 查看所有容器
docker ps #查看正在运行的容器
# 清理临时的、没有被使用的镜像文件。-a, --all: 删除所有没有用的镜像，而不仅仅是临时文件；
docker image prune 
```

## 拉取镜像
```shell
docker search mysql # 查看mysql相关镜像
docker pull mysql:5.7 # 拉取mysql镜像
docker image ls # 查看所有已下载镜像
```

## 构建镜像
```shell
# imageName 是镜像名称，1.0.0 是镜像的版本号或标签
docker build -t imageName:1.0.0 .
```

## 删除镜像
> [!example] 这里删除 MySQL 镜像

1. 删除之前需要查看这个镜像有没有被容器引用
```shell
docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
c4cd691d9f80        mysql:5.7           "docker-entrypoint.s…"   7 weeks ago         Up 12 days          0.0.0.0:3306->3306/tcp, 33060/tcp   mysql
```
这里可以看到MySQ正在被id为c4cd691d9f80 的容器引用
我们需要通过`docker stop c4cd691d9f80`或者`docker stop mysql`暂停这个容器

2. 查看MySQL镜像的id
```shell
docker images
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
mysql                   5.7                 f6509bac4980        3 months ago        373MB
```

3. 通过`image id`或则`repository`名字即可删除
```sh
docker rmi f6509bac4980 # 或者 docker rmi mysql
```

# Docker Compose
用于定义和运行多容器 Docker 应用程序的工具
可以使用 YML 文件来配置应用程序需要的所有服务
通过一个命令来创建并启动所有服务
文件可以分为三层
```yml
#第一层 版本号
version: "3.8"  #代表使用docker-compose项目的版本号
#第二层：services 服务配置
services:
  web:
    build: .
    ports:  #宿主机和容器的端口映射
      - "5000:5000"
    volumes:
      - .:/code
  redis:
     image: "redis:alpine"
# 第三层 其他配置 网络、卷、全局规划
```
