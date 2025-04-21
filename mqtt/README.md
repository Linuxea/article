# MQTT 入门实践

## 一、MQTT简介
MQTT（Message Queuing Telemetry Transport）是一个轻量级的发布-订阅模式的消息传输协议，特别适用于物联网（IoT）场景。它的工作原理建立在以下几个核心概念之上：

### Broker（消息代理）：

MQTT的核心组件，作为消息的中转站
负责接收所有消息，并将消息正确转发给订阅者
管理客户端的连接、会话和订阅关系

### Client（客户端）：

Publisher（发布者）：负责发送消息到特定主题
Subscriber（订阅者）：订阅感兴趣的主题以接收消息
一个客户端可以同时是发布者和订阅者

### Topic（主题）：

消息的传输通道，以层级结构组织
使用'/'分隔层级，如"home/livingroom/temperature"
支持通配符：'+'（单层）和'#'（多层）


## 二、Windows下的Mosquitto安装与配置
Mosquitto是最流行的开源MQTT broker之一，以下是在Windows系统上的安装步骤：

### 下载与安装：

1. 访问官方网站：https://mosquitto.org/download/
2. 下载Windows版本的安装包
3. 启动Mosquitto服务


## 三、Go MQTT客户端库介绍
本例中使用的是Eclipse Paho MQTT Go客户端库，它是最广泛使用的MQTT客户端库之一。

### 主要特性：

1. 支持MQTT 3.1和3.1.1版本
2. 提供同步和异步API
3. 自动重连机制
4. QoS 0,1,2 全支持
5. TLS/SSL安全连接
6. 持久会话支持

### 安装方法：
在 go 项目目录下
```bash
go get github.com/eclipse/paho.mqtt.golang
```

### 完整示例
```golang
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	mqtt "github.com/eclipse/paho.mqtt.golang"
)

// MQTTConfig 存储MQTT配置
type MQTTConfig struct {
	Broker   string
	ClientID string
	Topic    string
	QoS      byte
}

// 消息处理函数，当从订阅的主题收到消息时会被调用
var messagePubHandler mqtt.MessageHandler = func(client mqtt.Client, msg mqtt.Message) {
	log.Printf("Received message from topic: %s\n", msg.Topic())
	log.Printf("Message: %s\n", string(msg.Payload()))
}

// 连接建立成功时的处理函数
var connectHandler mqtt.OnConnectHandler = func(client mqtt.Client) {
	log.Println("Connected to MQTT broker")
}

// 连接丢失时的处理函数
var connectLostHandler mqtt.ConnectionLostHandler = func(client mqtt.Client, err error) {
	log.Printf("Connection lost: %v\n", err)
}

func main() {
	// 初始化日志
	log.SetFlags(log.LstdFlags | log.Lshortfile)

	// MQTT配置
	config := MQTTConfig{
		Broker:   "tcp://127.0.0.1:1883",
		ClientID: "simple-go-mqtt-client",
		Topic:    "test/topic",
		QoS:      1,
	}

	// 创建客户端配置
	opts := mqtt.NewClientOptions()
	opts.AddBroker(config.Broker)
	opts.SetClientID(config.ClientID)
	opts.SetKeepAlive(60 * time.Second)
	opts.SetPingTimeout(1 * time.Second)
	opts.SetConnectTimeout(5 * time.Second)
	opts.SetAutoReconnect(true)
	opts.SetMaxReconnectInterval(5 * time.Second)
	opts.OnConnect = connectHandler
	opts.OnConnectionLost = connectLostHandler
	opts.SetDefaultPublishHandler(messagePubHandler)

	// 创建客户端并连接
	client := mqtt.NewClient(opts)
	if token := client.Connect(); token.Wait() && token.Error() != nil {
		log.Fatalf("Failed to connect to MQTT broker: %v", token.Error())
	}

	// 订阅主题
	token := client.Subscribe(config.Topic, config.QoS, nil)
	if token.Wait() && token.Error() != nil {
		log.Printf("Failed to subscribe: %v", token.Error())
		client.Disconnect(250)
		return
	}
	log.Printf("Subscribed to topic: %s\n", config.Topic)

	// 创建上下文用于优雅退出
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// 处理系统信号
	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, syscall.SIGINT, syscall.SIGTERM)

	// 在后台发送消息
	go func() {
		for i := 1; i <= 10; i++ {
			select {
			case <-ctx.Done():
				return
			default:
				text := fmt.Sprintf("Hello MQTT from Go! Message %d", i)
				token = client.Publish(config.Topic, config.QoS, false, text)
				if token.Wait() && token.Error() != nil {
					log.Printf("Failed to publish message %d: %v", i, token.Error())
					continue
				}
				log.Printf("Published message: %s\n", text)
				time.Sleep(time.Second)
			}
		}
	}()

	// 等待中断信号
	<-sigChan
	log.Println("Shutting down...")

	// 清理订阅
	if token := client.Unsubscribe(config.Topic); token.Wait() && token.Error() != nil {
		log.Printf("Failed to unsubscribe: %v", token.Error())
	}

	// 断开连接
	client.Disconnect(250)
	log.Println("Disconnected from MQTT broker")
}
```


### 运行程序
```bash
go run main.go
```
程序将：

- 连接到本地MQTT服务器（localhost:1883）
- 订阅"test/topic"主题
- 每秒发送一条消息
- 同时接收来自该主题的消息
- 按Ctrl+C可优雅退出


### 最佳实践建议
1. 连接管理：
- 始终启用自动重连
- 设置合理的超时时间
- 实现连接状态监控

2. 消息处理：
- 使用适当的QoS级别
- 实现消息处理的错误恢复
- 考虑消息持久化需求

3. 安全性考虑：
- 在生产环境使用TLS/SSL
- 实现用户名密码认证
- 设置适当的访问控制

4. 性能优化：
- 合理设置QoS级别
- 适当的心跳间隔
- 注意消息大小和频率

## 总结
这个示例展示了如何使用Go语言实现一个功能完整的MQTT客户端，包含了实际应用中需要考虑的各个方面。通过这个例子，读者可以快速理解MQTT的工作原理，并在此基础上开发自己的MQTT应用。