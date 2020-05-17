---
title: 初识Docker容器
date: 2020-03-03
category: Java容器
tags: Docker
description: Docker是一个开源的应用容器引擎，开发者可以利用Docker打包自己的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。本书主要讲解了一些Docker的基本概念以及一些应用，整理了一些日常使用命令。
---

### 创建新Docker镜像

创建镜像主要有两种方式：

1）docker commit命令，基于运行的容器创建镜像

```sh
➜  ~ docker commit 6d065d8f2231 new-docker-image-name
➜  ~ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
my-hello-world      latest              35e9d9834200        1 minutes ago      13.3kB
hello-world         latest              bf756fb1ae65        4 months ago        13.3kB
```

2）编写DockerFile文件创建镜

Dockerfile文件输入以下内容

```dockerfile
FROM hello-world
RUN 
CMD ["/bin/bash", "-c", "my2 hello world!"] #hello-world必须有/bin/bash命令工具
```

```sh
➜  test_docker docker build -t my2-hello-world
➜  test_docker docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
my2-hello-world     latest              fc6038469ea4        10 seconds ago      13.3kB
my-hello-world      latest              35e9d9834200        5 minutes ago       13.3kB
hello-world         latest              bf756fb1ae65        4 months ago        13.3kB
```

### Docker常用命令

```sh
# 查看docker版本信息
docker info
# 查看镜像
docker images
# 查看容器
docker ps (-a) 
# 列出最后运行的容器，包括运行中和已经停止的
docker ps -l /docker ps -n 2
# 指定容器名称 first_example ,查找本地，pull remote ，启动
docker run -i -t --name=first_example ubuntu /bin/bash
docker restart first_example
docker stop first_example
# 创建后台进程 -d 参数
docker run --name daemon_example -d ubuntu /bin/sh -c "while true;do ..."
# 查看进程日志
docker logs daemon_example
docker logs --tail 10 daemon_example
docker logs --tail 0 -f -t daemon_example #不断输出,-t 输出时间
# 查看内部进程
docker top daemon_example
# 在容器额外启动新进程
# i 交互式
docker exec -t -i daemon_example /bin/bash
# d 后台式
docker exec -d daemon_example touch test_exec.txt
# 由于某种错误，进程停止，自动重启
docker run --restart=always --name first_example -d ubuntu /bin/sh -c "while true;do ..."
# inspect 深入容器获取更多信息
docker inspect first_example
# stop 停止容器
docker stop first_example
# rm 删除容器
docker rm first_example
# rm 删除所有容器
docker rm $(docker ps -aq)
# rmi 删除镜像之前先删除掉依赖该镜像到所有容器
docker rmi ubuntu_my_example
```

### 初识docker-compose

Dockerfile 可以让用户管理一个单独的应用容器；而 Compose 则允许用户在一个模板（YAML 格式）中定义一组相关联的应用容器（被称为一个 project，即项目），例如一个 Web 服务容器再加上后端的数据库服务容器等，就拿官网的 Python 例子来说，功能很简单，利用 redis 的 incr 的功能对页面的访问量进行统计。

docker-compose 的安装可以参考官网，如果安装了 Docker for Mac 或者 Docker for windows 则直接就存在了。

创建项目目录:

```sh
$ mkdir composetest
$ cd composetest
```

创建 Python 的程序 `app.py` ，功能就是利用 redis 的 incr 方法进行访问计数。

```python
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    redis.incr('hits')
    return 'Hello World! I have been seen %s times.' % redis.get('hits')

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```

由于 Python 依赖的 `flask`、`redis` 组件都需要另外安装，比如使用 `pip`来安装，单独设置一文件 `requirements.txt`，内容如下:

```
flask
redis
```

创建 service 依赖的第一个 image，`app.py` 程序的运行环境，利用 `Dockerfile` 来制作，内容如下:

```dockerfile
FROM python:2.7 #基于 python:2.7 镜像
ADD . /code  #将本地目录中的内容添加到 container 的 /code 目录下
WORKDIR /code  #设置程序工作目录为 /code
RUN pip install -r requirements.txt   #运行安装命令
CMD python app.py  #启动程序
```

`Dockerfile` 创建好就可以制作镜像了，运行`docker build -t compose/python_app .` ，成功后通过`docker images`查看即能看到:

```sh
docker images                                   
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
compose/python_app     latest              1a92fed00abd        59 minutes ago      680.4 MB
```

接下来制作 docker-compose 需要的配置文件 `docker-compose.yml`， version 要选择 2 ，1的版本很古老了，配置中可以看到创建了 2 个 service，`web` 和 `redis` ，各自有依赖的镜像，其中`web` 开放 container 的5000端口，并与 host 的5000端口应对，方便通过 `localhost:5000` 来访问， `volumes` 即将本地目录中的文件加载到容器的 `/code` 中，`depends_on` 表名 services `web` 是依赖另一个 service `redis`的，完整的配置如下:

```
version: '2'
  services:
    web:
      image: compose/python_app
      ports:
       - "5000:5000"
      volumes:
       - .:/code
      depends_on:
       - redis
    redis:
      image: redis
```

前提都搞定了，就差最后一步启动了，命令 `docker-compose up` ，成功后如下:

```
Attaching to composetestbypython_redis_1
redis_1  | 1:C 04 Nov 10:35:17.448 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
redis_1  |                 _._
redis_1  |            _.-``__ ''-._
redis_1  |       _.-``    `.  `_.  ''-._           Redis 3.2.5 (00000000/0) 64 bit
redis_1  |   .-`` .-```.  ```\/    _.,_ ''-._
redis_1  |  (    '      ,       .-`  | `,    )     Running in standalone mode
redis_1  |  |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
redis_1  |  |    `-._   `._    /     _.-'    |     PID: 1
redis_1  |   `-._    `-._  `-./  _.-'    _.-'
redis_1  |  |`-._`-._    `-.__.-'    _.-'_.-'|
redis_1  |  |    `-._`-._        _.-'_.-'    |           http://redis.io
redis_1  |   `-._    `-._`-.__.-'_.-'    _.-'
redis_1  |  |`-._`-._    `-.__.-'    _.-'_.-'|
redis_1  |  |    `-._`-._        _.-'_.-'    |
redis_1  |   `-._    `-._`-.__.-'_.-'    _.-'
redis_1  |       `-._    `-.__.-'    _.-'
redis_1  |           `-._        _.-'
redis_1  |               `-.__.-'
redis_1  |
redis_1  | 1:M 04 Nov 10:35:17.450 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
redis_1  | 1:M 04 Nov 10:35:17.450 # Server started, Redis version 3.2.5
redis_1  | 1:M 04 Nov 10:35:17.451 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
redis_1  | 1:M 04 Nov 10:35:17.451 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
redis_1  | 1:M 04 Nov 10:35:17.451 * The server is now ready to accept connections on port 6379
```

此时通过compose 的 ps 命令也能看到 `docker-compose ps` :

```
docker-compose ps                                                                       
          Name                          Command               State           Ports
---------------------------------------------------------------------------------------------
composetestbypython_redis_1   docker-entrypoint.sh redis ...   Up      6379/tcp
composetestbypython_web_1     /bin/sh -c python app.py         Up      0.0.0.0:5000->5000/tcp
```

ok ，在浏览器中访问 `http://localhost:5000` 就能看到最终的样子啦。

`docker-compose` 还有很多实用的命令，比如 logs、build、start、stop 等，可以通过 `docker-compose --help`来查看:

```sh
Define and run multi-container applications with Docker.

Usage:
  docker-compose [-f <arg>...] [options] [COMMAND] [ARGS...]
  docker-compose -h|--help

Options:
  -f, --file FILE             Specify an alternate compose file (default: docker-compose.yml)
  -p, --project-name NAME     Specify an alternate project name (default: directory name)
  --verbose                   Show more output
  -v, --version               Print version and exit
  -H, --host HOST             Daemon socket to connect to

  --tls                       Use TLS; implied by --tlsverify
  --tlscacert CA_PATH         Trust certs signed only by this CA
  --tlscert CLIENT_CERT_PATH  Path to TLS certificate file
  --tlskey TLS_KEY_PATH       Path to TLS key file
  --tlsverify                 Use TLS and verify the remote
  --skip-hostname-check       Don't check the daemon's hostname against the name specified
                              in the client certificate (for example if your docker host
                              is an IP address)

Commands:
  build              Build or rebuild services
  bundle             Generate a Docker bundle from the Compose file
  config             Validate and view the compose file
  create             Create services
  down               Stop and remove containers, networks, images, and volumes
  events             Receive real time events from containers
  exec               Execute a command in a running container
  help               Get help on a command
  kill               Kill containers
  logs               View output from containers
  pause              Pause services
  port               Print the public port for a port binding
  ps                 List containers
  pull               Pulls service images
  push               Push service images
  restart            Restart services
  rm                 Remove stopped containers
  run                Run a one-off command
  scale              Set number of containers for a service
  start              Start services
  stop               Stop services
  unpause            Unpause services
  up                 Create and start containers
  version            Show the Docker-Compose version information
```

更多 `docker-compose` 例子参考官网 doc 文档 :

- Docker-compose with Django : https://docs.docker.com/compose/django/
- Get started with Rails : https://docs.docker.com/compose/rails/
- Get started with WordPress : https://docs.docker.com/compose/wordpress/