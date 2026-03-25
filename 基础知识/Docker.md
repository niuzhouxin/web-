因为复现很多漏洞都需要Linux环境，虚拟环境，只用小皮面板明显不够用了，就学一下Docker怎么搭建环境。学习一下一个web题目是怎么用Docker搭建起来的。
## 基本操作
例如指令
```
docker run ubuntu:22.04 /bin/echo "Hello world"
```
参数解析
- docker:Docker的二进制执行文件。
- run: 与前面的docker组合来运行一个容器。
- ubuntu:22.04 指定要运行的镜像，Docker 首先从本地主机上查找镜像是否存在，如果不存在，Docker 就会从镜像仓库 Docker Hub 下载公共镜像。
- /bin/echo "Hello world": 在启动的容器里执行的命令
以上命令完整的意思可以解释为：Docker 以 ubuntu:22.04 镜像创建一个新容器，然后在容器里执行 bin/echo "Hello world"，然后输出结果。
#### 运行交互式容器
```
docker run -i -t ubuntu:22.04 /bin/bash
```
这样可以创建并进入容器内部。当出现`root@2c75640ee9a6:/#`说明进入容器内部。
参数解析：
- `-t`：在新容器内指定一个终端或伪终端。
- `-i`：允许你对容器的标准输入(STDIN) 进行交互。
进入后就可以执行指令了例如`ls`，输入`exit`可以退出。
#### 启动容器(后台模式)
使用以下命令创建一个以进程方式运行的容器
```
docker run -d ubuntu:22.04 /bin/sh -c "while true; do echo hello world; sleep 1; done"
```
- `-d`：表示在后台运行容器。
会输出一长串字符`b56763ce21b5dba77ce4408b18b8329ee80f4625082eaa8eb665f4608bba6590`
这就是容器ID，我们可以通过容器ID来查看容器内部正在进行什么。
```
docker ps
```
这可以列出所有容器
```
CONTAINER ID   IMAGE         COMMAND   ....
b56763ce21b5   ubuntu:22.04  "/bin/sh -c 'while t…"  ...
```
解释
- CONTAINER ID：容器ID
- IMAGE：使用的镜像。
- COMMAND：启动容器时运行的命令。
- CREATED：容器创建的时间。
- PORTS：端口映射情况。器的端口信息和使用的连接类型（tcp\udp）。
- STATUS：容器状态。
- NAMES：自动分配的容器名称。
在宿主机
```
docker logs b56763ce21b5
```
这样就可以查看容器内的标准输出，也就会输出那一大堆hello world
#### 停止容器
可以使用
```
docker stop b56763ce21b5
```
来停止容器，再`docker ps`那个容器就不见了。
也可以用NAMES来停止容器。
```
docker stop quizzical_almeida
```
#### 容器与镜像的关系
- **镜像（Image）**：容器的静态模板，包含了应用程序运行所需的所有依赖和文件。镜像是不可变的。
- **容器（Container）**：镜像的一个运行实例，具有自己的文件系统、进程、网络等，且是动态的。容器从镜像启动，并在运行时保持可变。
#### Docker常用指令
| 命令                  | 功能                                 | 示例                                       |
| ------------------- | ---------------------------------- | ---------------------------------------- |
| docker run          | 启动一个新的容器并运行命令                      | docker run -d ubuntu                     |
| docker ps           | 列出当前正在运行的容器                        | docker ps                                |
| docker ps -a        | 列出所有容器（包括已停止的容器）                   | docker ps -a                             |
| docker build        | 使用Dockerfile构建镜像                   | docker build -t my-image .               |
| docker images       | 列出本地存储的所有镜像                        | docker images                            |
| docker pull         | 从 Docker 仓库拉取镜像                    | docker pull ubuntu                       |
| docker push         | 将镜像推送到 Docker 仓库                   | docker push my-image                     |
| docker exec         | 在运行的容器中执行命令                        | docker exec -it 23dfd567926a bash        |
| docker stop         | 停止一个或多个容器                          | docker stop container_name               |
| docker start        | 启动已经停止的容器                          | docker start container_name              |
| docker restart      | 重启一个容器                             | docker restart container_name            |
| docker rm           | 删除一个或多个容器                          | docker rm container_name                 |
| docker rmi          | 删除一个或多个镜像                          | docker rmi my-image                      |
| docker logs         | 查看容器的日志                            | docker logs container_name               |
| docker inspect      | 获取容器或镜像的详细信息                       | docker inspect container_name            |
| docker exec -it     | 进入容器的交互式终端                         | docker exec -it container_name /bin/bash |
| docker network ls   | 列出所有 Docker 网络                     | docker network ls                        |
| docker volume ls    | 列出所有 Docker 卷                      | docker volume ls                         |
| docker-compose up   | 启动多容器应用（从 `docker-compose.yml` 文件） | docker-compose up                        |
| docker-compose down | 停止并删除由 `docker-compose` 启动的容器、网络等  | docker-compose down                      |
| docker info         | 显示 Docker 系统的详细信息                  | docker info                              |
| docker version      | 显示 Docker 客户端和守护进程的版本信息            | docker version                           |
| docker stats        | 显示容器的实时资源使用情况                      | docker stats                             |
| docker login        | 登录 Docker 仓库                       | docker login                             |
| docker logout       | 登出 Docker 仓库                       | docker logout                            |
常用选项说明
- `-d`：后台运行容器，例如 `docker run -d ubuntu`。
- `-it`：以交互式终端运行容器，例如 `docker exec -it container_name bash`。
- `-t`：为镜像指定标签，例如 `docker build -t my-image .`。
- `-p`：端口映射，例如`docker run -d -p 8085:80 --name pear_test pear`
#### 容器使用
**获取镜像**
如果我们本地没有 ubuntu 镜像，我们可以使用 docker pull 命令来载入 ubuntu 镜像：
```
docker pull ubuntu
```
**启动容器**
```
docker run -it ubuntu /bin/bash
```
## 题目组成
一般一个web题是由这些组成的
```
ctf
├── Dockerfile
├── docker-compose.yml（可选）
├── flag
├── start.sh
└── src # 网站源码
    └── index.php
```
## Dockerfile
最简单的Dockerfile
```Dockerfile
FROM php:7.4-apache
COPY src/ /var/www/html/
COPY flag /flag
WORKDIR /var/www/html
ENV FLAG=flag{Th1s_is_a_fake_flag}
RUN chmod 444 /flag
RUN a2enmod rewrite
EXPOSE 80
CMD ["apache2-foreground"]
```
其中
- `FROM php:7.4-apache`指定基础镜像，这个镜像Apache 已安装并配置好，PHP 已与 Apache 集成，默认网站目录：`/var/www/html` 一步到位。
- `COPY src/ /var/www/html/`把本地源码拷贝进容器。宿主机当前目录下的 `src` 文件夹内容，复制到容器的 `/var/www/html/`(网页根目录)
- 这样访问`http://ip:port/index.php`就可以执行源码。如果有多个文件，都可以访问`http://ip:port/phpinfo.php`
- `COPY flag /flag`把宿主机当前目录下的 `flag` 文件，复制到容器根目录 `/flag`
- `RUN chmod 444 /flag` 设置 flag 文件权限为“只读” `444 = r-- r-- r--`也可以设置为`400`只允许root读，这样就不得不提权了。
- `RUN a2enmod rewrite` 启用 Apache 的 rewrite 重写模块，实现伪静态 URL，路由转发，访问控制。（这个不是每道题都要用，但还是写上）
- `ENV FLAG=flag{Th1s_is_a_fake_flag}`添加环境变量。
- `EXPOSE 80` 声明容器对外暴露 80 端口
- `WORKDIR /var/www/html`设置工作目录
- `CMD ["apache2-foreground"]`默认启动apache
## 搭建环境
1.  `cd 题目路径`
2. `docker build -t pear .`  `-t pear`给镜像起名为`pear` `.`表示使用**当前目录**作为构建上下文 (但是这样不知道为什么一直失败，试了好多次，发现只有先`docker pull php:7.4-apache` ，再`docker build -t pear .`)
3. 再`docker run -d -p 8085:80 --name pear_test pear` 运行容器，`-d` 后台运行，`-p 8085:80`端口映射，宿主机的8085映射到容器的80。
4. `--name pear_test` 给容器起名
5. `pear` 镜像名。
6. 再访问`http://localhost:8085`就可以进入题目了。  
7. 可以用`docker exec -it pear_test bash`可以在控制台进入容器，方便调试。
8. 如果想重启容器，需要
```
docker stop pear_test
docker rm pear_test
docker build -t pear .
docker run -d -p 8085:80 --name pear_test pear
```
先写到这里，以后用到再补充。
## .dockerignore文件
执行`COPY src/ /var/www/html/`会拷贝目录的所有文件，如果有需要忽略的文件，就需要用到.dockerignore文件
编写.dockerignore文件，内容是
```
# .dockerignore 示例（注释用 # 开头） 
# 忽略 node_modules 目录（整目录） 
node_modules/ 
# 忽略 logs 目录 
logs/ 
# 忽略 .git 版本控制目录 
.git/ 
# 忽略单个文件 
.env 
temp.txt 
# 忽略所有 .log 后缀的文件 
*.log 
# 忽略所有以 temp 开头的文件/目录 
temp* 
# 例外：如果想忽略所有 .txt 但保留 important.txt 
*.txt 
!important.txt
```
## docker-compose.yml
后缀可以是yml或yaml
Dockerfile只可以用来构建单个镜像，在docker-compose.yml文件中，可以定义多个服务，每个服务可以包含一系列配置选项，例如镜像名称、容器端口、环境变量等。
文件格式
```yml
# 指定 compose 文件版本（推荐用 3.x，兼容性最好）
version: '3.8'

# 定义所有服务（每个服务对应一个容器）
services:
  # 服务名称（自定义，比如叫 my-app）
  服务名:
    # 【核心】自动构建镜像的配置
    build:
      context: .          # 构建上下文（Dockerfile 所在目录，. 表示当前目录）
      dockerfile: Dockerfile  # 指定 Dockerfile 文件名（默认就是 Dockerfile，可省略）
    # 【核心】端口映射（替代 docker run -p）
    ports:
      - "主机端口1:容器端口1"  # 比如 "8080:80"
      - "主机端口2:容器端口2"  # 可选，支持多端口映射
    # 可选：给容器自定义名称（替代 docker run --name）
    container_name: 自定义容器名
    # 可选：容器后台运行（默认就是，对应 docker run -d）
    restart: always  # 容器退出时自动重启（可选，推荐加）
```
Compose 文件是一个 YAML 文件，用于定义 Docker 应用程序的多个容器化服务。这个文件通常用于定义服务、网络、卷、配置和秘密等。

docker-compose.yml文件有以下6个顶级元素：

- **version (可选)**
描述：指定 Compose 文件的版本。这有助于确保 Compose 工具与您的文件格式兼容。
用途：当工具或库进行更新时，版本号可以帮助确保您的 Compose 文件与新版本的工具或库兼容。
- services (必需)
描述：定义应用程序的各个服务。每个服务是一个独立的 Docker 容器，包含运行应用程序所需的配置信息。
用途：在 Compose 文件中，您可以为应用程序的不同部分定义多个服务。例如，一个 Web 服务器和一个数据库可以作为两个不同的服务来管理。
- networks (可选)
描述：定义自定义网络，以便容器可以相互通信。
用途：默认情况下，Compose 会为您的应用创建一个网络，但您也可以定义自己的网络，以便容器可以与外部世界或其他容器通信。
- volumes (可选)
描述：定义数据卷，以便持久化存储数据或共享数据。
用途：数据卷允许您在容器之间共享和持久化数据。这对于确保数据的一致性和持久性非常有用。
- configs (可选)
描述：定义配置，这些配置可以在服务中使用，但不应该直接在 Compose 文件中硬编码。
用途：这是一个相对较新的功能，允许您将敏感信息（如密码、API 密钥等）从 Compose 文件中移出，以增加安全性。这些配置可以在运行时注入到服务中。
- secrets (可选)
描述：类似于 Configs，Secrets 也用于定义敏感信息，但它们是专为敏感数据设计的，并具有特定的管理功能。
用途：Secrets 是专为存储敏感信息而设计的，例如密码、API 密钥等。它们可以与 Configs 结合使用，以提供更完整和安全的解决方案来管理敏感数据。
这些配置项使 Docker Compose 成为一个功能强大的工具，用于定义、部署和管理复杂的 Docker 应用程序。
例子
```yml
version: '3'
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    networks:
      - app-net
    command: ["nginx", "-g", "daemon off;"]
  redis:
    image: redis:latest
    networks:
      - app-net
    command: ["redis-server"]
networks:
  app-net:

```
这个 `docker-compose.yml` 文件定义了两个服务：web 和 redis。每个服务都有自己的配置。
- web 服务使用 nginx:alpine 镜像，这是 Nginx 的一个轻量级版本。它将容器的 80 端口映射到主机的 8080 端口。它还将当前目录下的 nginx.conf 文件挂载到容器的 /etc/nginx/nginx.conf，这样我们就可以在主机上修改 Nginx 的配置。最后，它使用 daemon off; 命令来确保 Nginx 在容器退出时停止运行。
- `redis` 服务使用最新的 Redis 镜像。它连接到名为 `app-net` 的网络，并使用 `redis-server` 命令启动 Redis。
- `app-net` 网络允许这两个服务相互通信。


## 参考
https://www.runoob.com/docker/docker-tutorial.html
https://blog.csdn.net/wlddhj/article/details/135481281