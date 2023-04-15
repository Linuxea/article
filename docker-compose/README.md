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

## docker-compose.yml






