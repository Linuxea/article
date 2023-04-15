#Docker Compose

## Preview

Docker 是一种强大的容器技术，它的出现改变了应用程序开发和部署的方式。

而今天提到的 Docker-Compose 是基于 Docker 提供的强大管理工具。

顾名思义，将Docker Compose与设计模式中的组合模式进行类比，组合模式是一种结构型设计模式，它允许你将对象组合成树形结构以表示“整体/部分”的层次结构。

组合模式能让客户端以统一的方式处理单个对象和对象组合。

在这种情况下，我们可以将Docker容器看作是单个对象，而Docker Compose则可以看作是组合这些容器的工具。

它以一种统一的方式处理和组织多个Docker容器。通过使用Docker Compose，可以更加高效地管理复杂的多容器应用程序，提高开发和运维团队的协作效率。


这篇文章中，我们尝试用 Docker-Compose 来搭建一个简单的开发环境，包含以下中间件及对应的版本(image:tag)
- redis:latest
- mysql:8.0.23
- xxljob:2.1.2
- rabbitmq:3.8.0

## 定义配置文件 docker-compose.yml

docker-compose.yml 是默认的配置文件名，它是一个用于定义和配置多容器Docker应用程序的YAML文件。

这里以配置 redis 最新版本的服务为例:
```yml
version: '3.8'
services:
  linuxea-redis:
    image: redis:latest
    container_name: linuxea-redis
    expose:
      - "6379"
    ports:
      - 6379:6379
    volumes:
      - redis-data:/data

volumes:
  redis-data:
```

- version: version指令在docker-compose.yml文件中用于指定Docker Compose的文件格式版本。不同版本的Docker Compose文件格式可能支持不同的功能和选项。通过设置该指令确保Docker Compose工具正确地解析并处理配置文件。
- services: 定义一组服务，这也是 Docker-Compose 的核心，批量定义与管理多个服务
  - linuxea-redis: 声明一个服务，使用的自定义名称
    - image: 该服务使用的镜像与tag
    - container_name: Docker 容器的名称，即 docker ps 名称一栏展示
    - expose: 声明端口，一种约定，让开发者彼此间了解到该服务所使用的端口
    - ports: 暴露端口，将本地端口映射到容器内的端口。是真正的暴露端口指令
    - volumes: redis-data 卷被映射到容器的 /data 目录，将容器数据持久化
- volumes: 定义一组数据卷，提供服务直接使用

为了体现 Docker Compose 的优势，我们再添加一个服务 mysql，最终配置如下:
```yaml
version: '3.8'
services:
  linuxea-redis:
    image: redis:latest
    container_name: linuxea-redis
    expose:
      - "6379"
    ports:
      - 6379:6379
    volumes:
      - redis-data:/data

  linuxea-mysql:
    image: mysql:8.0.23
    container_name: linuxea-mysql
    expose:
      - "3306"
    environment:
      MYSQL_ROOT_PASSWORD: mypassword
    volumes:
      - mysql-data:/var/lib/mysql

volumes:
  redis-data:
  mysql-data:
```


## Docker Compose 命令

我们尝试将以上的文件保存到 docker-compose.yml 文件中，并在该目录下运行 Docker Compose 的启动命令。
![docker-compose-up.png](docker-compose-up.png 'docker-compose-up.png')

我们使用了 `docker-compose -f ./docker-compose.yml up` 来启动配置中相关服务。
- docker-compose 是主要的 Docker Compose 的命令
- -f 用来指定使用的配置文件，当存在符合默认名称的配置文件时，则不需要显式指定，此处为尽可能覆盖讲解则显式指定。（关于默认配置名称，在不同的 Docker-Compose 版本中是有差异的，如 compose.yaml, compose.yml, docker-compose.yaml, docker-compose.yml 等多个版本）
- up: 创建并启动容器

最终 docker-compose.yml 配置的服务（们）都启动了并以前台方式运行。

>如果需要后台方式运行，可以使用 `docker-compose -f ./docker-compose.yml up -d`。


可以看到，我们配置的服务全部成功启动，通过 Docker Compose 可以实现一次性管理多个容器。
我们还可以细粒度控制单个服务。

```bash
➜  docker-compose -f ./docker-compose.yml up linuxea-redis
```

![docker-compose-up-redis.png](docker-compose-up-redis.png 'docker-compose-up-redis.png')

通过这种方式可以创建并启动指定容器 `linuxea-redis`。

类似的 docker-compose 命令还有
批量查看服务的日志
```
➜  docker-compose -f ./docker-compose.yml logs
```

批量查看服务的运行状态
```
➜  docker-compose -f ./docker-compose.yml ps
```

批量删除停止的容器
```
➜  docker-compose -f ./docker-compose.yml rm
```

批量停止与删除容器
```bash
➜  docker-compose -f ./docker-compose.yml down
```

更多的命令可以查阅 `docker-compose --help`
```bash
Commands:
  build       Build or rebuild services
  config      Parse, resolve and render compose file in canonical format
  cp          Copy files/folders between a service container and the local filesystem
  create      Creates containers for a service.
  down        Stop and remove containers, networks
  events      Receive real time events from containers.
  exec        Execute a command in a running container.
  images      List images used by the created containers
  kill        Force stop service containers.
  logs        View output from containers
  ls          List running compose projects
  pause       Pause services
  port        Print the public port for a port binding.
  ps          List containers
  pull        Pull service images
  push        Push service images
  restart     Restart service containers
  rm          Removes stopped service containers
  run         Run a one-off command on a service.
  start       Start services
  stop        Stop services
  top         Display the running processes
  unpause     Unpause services
  up          Create and start containers
  version     Show the Docker Compose version information
```

现在我们大概清楚 docker-compose 所提供的批量容器管理能力，通过指定服务，docker-compose 也能够实现对单个服务的管理。


## 搭建简易的开发环境

回到开头，我们提到的要搭建的开发环境：
- redis:latest
- mysql:8.0.23
- xxljob:2.1.2
- rabbitmq:3.8.0

使用 docker compose 能够轻易完成这些中间件的创建与管理。

```yml
version: '3.8'
services:
  linuxea-redis:
    image: redis:latest
    container_name: linuxea-redis
    expose:
      - "6379"
    volumes:
      - redis-data:/data

  linuxea-mysql:
    image: mysql:8.0.23
    container_name: linuxea-mysql
    expose:
      - "3306"
    environment:
      MYSQL_ROOT_PASSWORD: mypassword
    volumes:
      - mysql-data:/var/lib/mysql

  linuxea-xxljob:
    image: xuxueli/xxl-job-admin:2.1.2
    container_name: linuxea-xxljob
    ports:
      - 8080:8080
    volumes:
      - xxljob-data:/data/applogs
    environment:
      - spring.datasource.url=jdbc:mysql://linuxea-mysql:3306/xxl_job
      - spring.datasource.username=root
      - spring.datasource.password=mypassword
    depends_on:
      - linuxea-mysql

  linuxea-rabbitmq:
    image: rabbitmq:3.8.0
    container_name: linuxea-rabbitmq
    expose:
      - "5672"
      - "15672"
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq

volumes:
  redis-data:
  mysql-data:
  xxljob-data:
  rabbitmq-data:
```

关于这份整合后的配置文件，需要再提到几个新指令：
- environment: 通过设置环境变量的值，提供给容器内部使用
- depends_on: 用来管理服务之间的依赖关系，会影响服务之间的创建与启动顺序
- 在同一个docker-compose.yml文件中定义的服务，默认会被创建在同一个网络中。Docker会自动为这些服务分配一个内部的DNS解析器。这使得这些服务之间能够通过容器名称进行相互访问，而无需暴露宿主机上的端口。这种内部网络通信模式提供了更好的安全性和隔离性。因此，我们可以直接在 `linuxea-xxljob` 容器内部直接使用 `linuxea-mysql` 来访问数据库所在容器


接下来，我们重新启用这份配置文件:
![docker-compose-up-test.png](docker-compose-up-test.png 'docker-compose-up-test.png')

- ➜  docker-compose -f ./docker-compose.yml up -d    (以后台形式创建并启动所有服务)
- ➜  docker-compose-article docker-compose -f ./docker-compose.yml ps （查看所有服务运行状况）


使用 Docker-Desktop 可以更直观地查看 Docker-Compose 对多个服务之间的整合：
![docker-compose-up-desktop.png](docker-compose-up-desktop.png 'docker-compose-up-desktop.png')