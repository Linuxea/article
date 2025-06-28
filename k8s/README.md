# **从 Go 服务到 K8s 部署：云原生实践指南**

在现代云原生应用开发中，Go 语言因其高性能、简洁的语法和强大的并发能力而备受青睐。而 Docker 和 Kubernetes (K8s) 则分别是容器化和容器编排领域的标准。本文将完成一个完整的流程：编写一个基础的 Go Web 服务，使用 Docker 将其容器化，并最终将其部署到 Kubernetes 集群中。

### **第一步：构建基础 Go Web 服务**

我们的第一步是创建一个简单的 HTTP 服务器。它将监听一个端口，并在收到请求时返回一条简单的 “Hello, World\!” 消息。我们将使用 Go 语言内置的 net/http 包。

**main.go**

创建一个名为 main.go 的文件，并添加以下代码：

```golang
package main

import (  
	"fmt"  
	"log"  
	"net/http"  
)

func handler(w http.ResponseWriter, r *http.Request) {  
	// 向响应写入一条欢迎消息  
	fmt.Fprintf(w, "你好，世界！这是一个来自 Go 服务的响应。")  
}

func main() {  
	// 注册一个处理函数来处理所有指向根路径 ("/") 的请求  
	http.HandleFunc("/", handler)

	// 定义服务监听的端口  
	port := "8080"  
	fmt.Printf("服务启动，监听端口 %s\\n", port)

	// 启动 HTTP 服务  
	// log.Fatal 会在服务启动失败时记录错误并退出程序  
	if err := http.ListenAndServe(":"+port, nil); err != nil {  
		log.Fatal(err)  
	}  
}
```

**本地运行与测试**

在您的终端中，使用以下命令来运行这个服务：

```bash
go run main.go
```

你会看到 "服务启动，监听端口 8080" 的输出。现在，打开浏览器或使用 curl 访问 http://localhost:8080，您将收到我们预设的响应消息。

### **第二步：应用容器化 (Docker)**

服务可以运行后，下一步是将其打包成一个 Docker 镜像。这样可以确保它在任何环境中都能以相同的方式运行。我们将使用**多阶段构建 (Multi-stage build)** 的方式来创建 Dockerfile，这是一种最佳实践，可以显著减小最终镜像的体积。

**Dockerfile**

在项目根目录下创建一个名为 Dockerfile 的文件，并添加以下内容：

```Dockerfile
# --- 第一阶段：构建阶段 ---  
# 使用官方的 Go 镜像作为构建环境  
# 我们给这个阶段命名为 "builder"  
FROM golang:1.24-alpine AS builder

# 设置工作目录  
WORKDIR /app

# 复制 Go 模块文件  
# 这可以利用 Docker 的层缓存机制，只有在 go.mod 或 go.sum 改变时才重新下载依赖  
COPY go.mod ./
RUN go mod download

# 复制所有源代码  
COPY . .

# 构建 Go 应用  
# -o app 指定输出文件名为 "app"  
# CGO\_ENABLED=0 禁用了 CGO，这有助于构建一个静态链接的二进制文件  
# GOOS=linux 指定目标操作系统为 Linux  
RUN CGO_ENABLED=0 GOOS=linux go build -o app .

# --- 第二阶段：运行阶段 ---  
# 使用一个非常小的基础镜像，如 alpine，来减小最终镜像的体积  
FROM alpine:latest

# 设置工作目录  
WORKDIR /app

# 从 "builder" 阶段复制编译好的二进制文件  
COPY --from=builder /app/app .

# 暴露服务运行的端口  
EXPOSE 8080

# 定义容器启动时执行的命令  
CMD ["./app"]
```

**构建、推送和运行 Docker 镜像**

1. **构建镜像**: 在包含 Dockerfile 和 main.go 的目录中，运行以下命令。我们将镜像命名为 go-k8s-app。  
   <code>docker build \-t go-k8s-app .</code>

2. **启动本地镜像仓库**: 为了让 Kubernetes 能够拉取到我们构建的镜像，我们需要一个镜像仓库。出于演示目的，我们可以在本地启动一个 Docker Registry 容器。  
   <code>docker run -d -p 5000:5000 --restart=always --name registry registry:2</code>

   这个命令会在后台启动一个名为 registry 的容器，并将容器的 5000 端口映射到主机的 5000 端口。  
3. **标记并推送到本地仓库**: 构建好的镜像需要被正确地标记（tag），以便推送到我们的本地仓库。  
   # 标记镜像，指向本地仓库地址 localhost:5000  
   `docker tag go-k8s-app localhost:5000/go-k8s-app`

   # 推送镜像到本地仓库  
   `docker push localhost:5000/go-k8s-app`

4. **本地运行容器 (验证)**: 在推送到仓库后，我们仍然可以像之前一样在本地运行容器进行验证。  
   `docker run -p 8080:8080 localhost:5000/go-k8s-app`

   注意此时我们使用了带有仓库地址的完整镜像名。再次访问 http://localhost:8080 能看到一样的结果。

### **第三步：部署到 Kubernetes (K8s)**

现在我们有了一个位于镜像仓库中的 Docker 镜像，是时候将它部署到 Kubernetes 集群了。为此，我们需要定义几个 K8s 资源：`Deployment``、Service` 和 `Ingress`。

**1 Deployment: 管理应用实例**

Deployment 负责创建和维护我们应用的多个副本 (Pod)，确保应用的可用性和可扩展性。

**deployment.yaml**

```yaml
apiVersion: apps/v1  
kind: Deployment  
metadata:  
  name: go-app-deployment  
spec:  
  replicas: 3 # 指定运行 3 个应用的副本  
  selector:  
    matchLabels:  
      app: go-k8s-app  
  template:  
    metadata:  
      labels:  
        app: go-k8s-app  
    spec:  
      containers:  
      - name: go-k8s-app  
        # 重要：更新镜像地址为我们的本地仓库地址  
        image: localhost:5000/go-k8s-app   
        # 在真实环境中，应使用 docker.io/your-username/go-k8s-app 这样的公共地址  
        # 或者是一个 K8s 集群可以访问到的私有仓库地址  
        imagePullPolicy: Always # 建议设为 Always，确保每次都尝试拉取  
        ports:  
        - containerPort: 8080
```

**2 Service: 暴露应用**

```yaml
Deployment 创建的 Pods 是有生命周期的，它们的 IP 地址会变化。Service 提供了一个稳定的内部 IP 地址和 DNS 名称，用于访问这些 Pods。

**service.yaml**

apiVersion: v1  
kind: Service  
metadata:  
  name: go-app-service  
spec:  
  selector:  
    app: go-k8s-app # 选择标签为 app: go-k8s-app 的 Pods  
  ports:  
    - protocol: TCP  
      port: 80 # Service 监听的端口  
      targetPort: 8080 # 转发到 Pod 的端口  
  type: ClusterIP # 只在集群内部暴露服务
```

**3 Ingress: 允许外部访问**

Service 默认只在集群内部可访问。为了从外部互联网访问我们的服务，我们需要一个 Ingress。Ingress 负责管理外部访问的路由规则，例如基于主机名或路径的路由。

**ingress.yaml**

```yaml
apiVersion: networking.k8s.io/v1  
kind: Ingress  
metadata:  
  name: go-app-ingress  
spec:  
  rules:  
  - host: my-go-app.example.com # 您希望用来访问服务的域名  
    http:  
      paths:  
      - path: /  
        pathType: Prefix  
        backend:  
          service:  
            name: go-app-service # 将流量转发到 go-app-service  
            port:  
              number: 80 # Service 监听的端口
```

**部署到集群**

假设你已经配置好了 kubectl 并连接到了一个 K8s 集群，现在可以按顺序应用这些配置文件：

> 本地使用 minikube 搭建一个单节点 k8s 环境

```bash
# 应用 Deployment  
kubectl apply -f deployment.yaml

# 应用 Service  
kubectl apply -f service.yaml

# 应用 Ingress  
kubectl apply -f ingress.yaml
```

部署完成后，您可以检查状态：

```bash
# 查看 Pods 是否正在运行  
kubectl get pods

# 查看 Service 是否已创建  
kubectl get service

# 查看 Ingress 是否配置成功  
kubectl get ingress
```

最后，您需要配置您的 DNS 或本地 hosts 文件，将 my-go-app.example.com 指向您的 Ingress Controller 的 IP 地址。完成后，通过访问 http://my-go-app.example.com，您的请求将被 Ingress Controller 接收，然后转发到 go-app-service，并最终到达我们部署的 Go 应用 Pods 中的一个。

### **总结**

恭喜您！您已经成功地完成了一个完整的云原生应用部署流程。我们回顾一下：

1. 用 Go 构建了一个简单的 Web 服务。  
2. 使用 Docker 和多阶段构建技术将其容器化，并**推送到了一个本地镜像仓库**。  
3. 通过 Kubernetes 的 Deployment、Service 和 Ingress 资源，将应用部署为一个高可用的、可从外部访问的服务。

这个流程是现代软件开发和运维的核心实践，掌握它将为您的云原生之旅打下坚实的基础。