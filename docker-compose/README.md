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



## Docker Compose 命令

