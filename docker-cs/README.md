# Docker 远程连接的本质：从 CS 架构到三种连接方式的深度解析

对于经常在本地使用 docker 的你来说，第一次听说 Docker 支持"远程连接"时，可能会觉得陌生。但是， Docker 从设计之初就是一个天然支持远程操作的系统。为了理解这一点，我们回到 Docker 的架构根基——客户端-服务器模型。

## 1. Docker 的 CS 架构：一切远程连接的基础

Docker 采用了经典的客户端-服务器（Client-Server，简称 CS）架构。在这个架构中，Docker daemon（守护进程，也就是 dockerd）扮演服务器的角色。它是真正的执行者，负责管理镜像、容器、网络、卷等所有 Docker 资源。daemon 持续在后台运行，等待接收指令并执行实际的容器操作。
```bash
> ps aux | grep dockerd
```
输出：
```output
root     2720028  0.0  3.9 2194164 80120 ?       Ssl  Dec10   0:54 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
```

而我们日常在命令行中使用的 `docker` 命令，实际上是客户端程序。这个客户端的职责是充当用户和 daemon 之间的翻译官。当你输入 `docker run nginx` 时，客户端会把这个命令翻译成符合 Docker API 规范的 HTTP 请求，发送给 daemon，然后等待 daemon 执行完成并返回结果。

cs 架构的优势在于客户端和服务器的解耦。客户端只需要知道如何与 daemon 通信，而不需要关心 daemon 内部如何工作。（web 开发同学想必深有体会）

这种解耦带来了灵活性：
- 客户端可以是命令行工具 (docker cli)，也可以是图形界面程序(docker desktop)，甚至可以是其他编程语言写的脚本；
- daemon 可以运行在本地，也可以运行在远程服务器上，甚至可以运行在云端！

更重要的是，这个架构意味着"本地操作"和"远程操作"在本质上没有区别。即使你在本地使用 Docker，每次执行 docker 命令时，客户端也是通过某种通信机制向 daemon 发送请求。而所谓的"远程连接"，只不过是改变了客户端与 daemon 之间的通信渠道。

接下来就是我们需要了解的三种不同的通信渠道。

## 2. Unix Socket：本地通信的高效方式

在 Linux 系统上，当你刚安装完 Docker 时，默认的通信方式是通过 Unix domain socket。这是一个特殊的文件，通常位于 `/var/run/docker.sock`。

```bash
> docker context ls
```
输出：
```output
NAME        DESCRIPTION                               DOCKER ENDPOINT               ERROR
default *   Current DOCKER_HOST based configuration   unix:///var/run/docker.sock   
```

Unix socket 是 Linux 提供的一种进程间通信（IPC）机制，专门用于同一台机器上的进程之间进行通信。相比网络 socket，Unix socket 不需要经过网络协议栈的处理，避免了 TCP/IP 协议带来的开销，因此在本地通信场景下更加高效。

当执行 `docker ps` 查看容器列表时，实际的通信流程是这样的：docker 客户端程序启动后，首先读取配置文件或环境变量，确定 daemon 的位置。在默认配置下，它会尝试连接 `/var/run/docker.sock` 这个文件。客户端通过这个 socket 文件向 daemon 发送一个 HTTP GET 请求，请求路径类似于 `/containers/json`。daemon 接收到请求后，查询当前运行的容器信息，将结果以 JSON 格式通过同一个 socket 返回给客户端。客户端收到 JSON 数据后，将其解析并格式化成我们在终端看到的表格形式。

这个过程虽然客户端和服务器都在同一台机器上，但`其通信机制`和远程连接完全一致，都是通过标准的 HTTP 协议进行。Docker API 本身就是 RESTful 风格的 HTTP API，无论底层使用什么传输方式，API 的格式都是一样的。

## 3. TCP 连接：让 Docker 走向网络

Unix socket 虽然高效，但有个明显的局限：它只能用于本地通信。如果你想从自己的笔记本管理远程服务器上的 Docker，Unix socket 就无能为力了。这时候就需要让 Docker daemon 监听网络端口，接受来自网络的连接请求。

Docker daemon 可以配置为监听 TCP 端口。传统的做法是在 daemon 启动时指定监听地址，比如 `dockerd -H tcp://0.0.0.0:2375`。这条命令告诉 daemon："除了默认的 Unix socket，也监听所有网络接口的 2375 端口"。一旦 daemon 开始监听这个端口，任何能够访问这台机器 2375 端口的客户端都可以连接并操作 Docker。

> 还可以通过配置文件 /etc/docker/daemon.json 方式去开启监听 tcp 端口

客户端要连接远程 daemon 也很简单，只需要通过 `-H` 参数指定 daemon 的地址即可，比如 `docker -H tcp://192.168.1.100:2375 ps`。这条命令会让客户端连接到 192.168.1.100 这台机器的 2375 端口，而不是使用本地的 Unix socket。

从通信内容来看，通过 TCP 连接发送的请求和通过 Unix socket 发送的请求完全一样，都是标准的 HTTP 请求。唯一的区别是传输通道变了：Unix socket 使用的是本地文件系统的特殊文件，而 TCP 使用的是网络套接字。

但是 TCP 连接带来的便利性也伴随着严重的安全风险。2375 端口的 TCP 连接默认是明文传输，没有任何加密和认证机制。这意味着任何能够访问这个端口的人都可以完全控制你的 Docker 环境，包括启动恶意容器、访问敏感数据、甚至通过容器逃逸获取宿主机权限。

为了解决安全问题，Docker 支持通过 TLS（传输层安全协议）对 TCP 连接进行加密和认证。daemon 可以配置为监听 2376 端口（TLS 的默认端口），并要求客户端提供证书进行双向认证。

## 4. SSH 连接：站在巨人肩膀上的安全方案

相比 TLS ，我们现在有了更加开箱即用的方案：ssh 连接。

在需要远程连接 Docker 的场景中，还有一种既安全又相对简单的方式：通过 SSH 连接。这种方式充分利用了 SSH 这个已经非常成熟的远程连接协议，而不是重新发明轮子。
```bash
  Host 43.153.xxx.xxx
    HostName 43.153.xxx.xxx
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
```

> 推荐通过密钥的认证方式，避免频繁输入密码

虽然也是基于 tcp 协议，SSH 连接的工作机制和前两种方式有本质区别。当使用 Unix socket 或 TCP 连接时，是在直接改变 Docker daemon 的监听方式，让 daemon 去适应不同的连接需求。而 SSH 方式则不同，它不需要改变 daemon 的配置，daemon 可以仍然只监听本地的 Unix socket！

那么 SSH 方式是如何实现远程访问的呢？它的过程是这样的：当配置 Docker 客户端使用 SSH 连接时，客户端会先通过 SSH 协议连接到远程主机，建立一个加密的 SSH 隧道。然后，客户端通过这个隧道访问远程主机上的本地 Unix socket，也就是 `/var/run/docker.sock`。从 daemon 的角度看，它接收到的仍然是来自本地 Unix socket 的请求，完全感觉不到这个请求实际上来自远程机器。

从架构层面看，SSH 方式实际上是在 Docker 的 CS 架构之外又套了一层 SSH 的 CS 架构。SSH 客户端和 SSH 服务器之间形成了第一层的 CS 通信，而在这个通信通道内部，Docker 客户端再通过 Unix socket 与 Docker daemon 进行第二层的 CS 通信。

SSH 方式的另一个优势是它不需要暴露任何额外的网络端口。如果你的服务器已经开放了 SSH 端口用于远程管理（这在几乎所有 Linux 服务器上都是标配），那么你可以直接利用这个端口来进行 Docker 远程操作，不需要再开放 2375 或 2376 这些 Docker 专用端口。这不仅减少了攻击面，也简化了防火墙配置。

## 5. Docker Context：优雅地管理多个连接

开始使用 Docker 远程连接时，会发现每次都要通过 `-H` 参数指定连接地址是件麻烦的事情。
```bash
docker -H "ssh://ubuntu@43.153.xxx.xx" ps
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

docker -H "ssh://ubuntu@43.153.xxx.xx" run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.
... ...
```

如果你需要管理多台服务器，每台服务器可能使用不同的连接方式（有的用 TCP，有的用 SSH），记住这些配置信息会变得越来越困难。Docker Context 功能就是为了解决这个问题而设计的。

Docker Context 允许你为不同的 Docker 环境保存配置信息，并通过简单的名称来切换。你可以把 context 理解为一个配置文件集合，它保存了连接某个 Docker daemon 所需的所有信息，包括连接方式、地址、认证凭据等。

创建一个 context 的过程很直观。比如你想为一台使用 SSH 连接的远程服务器创建 context，可以执行类似这样的命令，Docker 会把这个配置保存下来。之后，当你想操作这台远程服务器上的 Docker 时，只需要切换到这个 context，之后的所有 docker 命令都会自动使用这个 context 的配置，不需要每次都指定 `-H` 参数。

创建第一个context：
```bash
> docker context create remote_1 --docker "host=ssh://ubuntu@43.153.195.48"
remote_1
Successfully created context "remote_1"
```
切换使用 context：
```bash
> docker context use remote_1
remote_1
Current context is now "remote_1"
```
接下来一直都是在同一个 context 环境下运行：
```bash
> docker run --rm ubuntu echo 'hell ub'

Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
20043066d3d5: Pull complete
Digest: sha256:c35e29c9450151419d9448b0fd75374fec4fff364a27f176fb458d472dfc9e54
Status: Downloaded newer image for ubuntu:latest
hell ub
```

创建与使用第二个context：
```bash
> docker context create remote_2 --docker "host=ssh://ubuntu@43.153.xxx.xx"
remote_2
Successfully created context "remote_2"
> docker context use remote_2
remote_2
Current context is now "remote_2"
> docker run --rm ubuntu echo 'hell ub2'
hell ub2
```


## 6. 三种连接方式的适用场景

了解这三种连接方式的原理后，总结一下它们各自的适用场景吧。

Unix socket 是本地使用 Docker 的最佳选择。它性能最好，配置最简单，也是 Docker 的默认方式，基本上我们都是在使用这种方式。只要你是在 Docker daemon 所在的同一台机器上操作，就应该使用 Unix socket。

TCP 连接适合需要远程访问但对安全性要求不高的场景，比如在完全隔离的内网环境中进行测试。如果确实需要在生产环境使用 TCP 连接，务必配置 TLS 加密和证书认证。

SSH 连接是远程访问 Docker 的推荐方式，特别是在生产环境或需要跨越公网连接的场景中。它的安全性得到了充分验证，配置相对简单（特别是当服务器已经配置好 SSH 访问时），而且不需要暴露额外的端口。

## 结语

Docker 的远程连接能力是其基础的 CS 架构的自然结果。客户端和服务器的分离设计，让 Docker 具备了灵活的连接能力。Unix socket、TCP 连接和 SSH 连接这三种方式，只是在为客户端和服务器之间的通信提供不同的传输通道，它们在安全性、便利性和性能之间做出了不同的权衡。