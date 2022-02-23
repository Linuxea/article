# DNS Deep Dive


## DNS Server

In real world, when you watching news, making online orders, downloading files or listening to live broadcasting, you need to visit the website by domain names, for example, youtube.com, google.com, ..etc. You must also remember the name of these website, not their IP addresses, because IP addresses are very difficult to remember, compare to website names. Therefore you need an website/domain name address book, which the the DNS server.


在现实世界上网冲浪，当你观看新闻，购物，下载文件或者听广播时，你需要通过域名来访问网站，比如 youtube.com, google.com 等等。

你必须记住它们的网址而不是 IP，因为相比下 IP 地址并不容易被记住。

因此你需要一个域名地址簿，也就 DNS 服务器。



DNS is very important in everyday life. Everyone online needs to access it, but at the same time, this is also a very big challenge for it. If the DNS server fails, the entire internet will be down.



DNS 在日常生活非常重要。

每一次上网冲浪都需要对其访问，同时，这也是一个巨大的挑战。

如果 DNS 服务宕机，整个互联网世界都会下线。

In addition, people who surf the Internet are distributed all over the world, if everyone goes to the same place to access a certain server, the delay will be very large. Therefore, the DNS server must be set to be highly available, highly concurrent and distributed.

另外，上网冲浪的人分布世界各地，如果每个人访问同个地点的一个中心服务，延迟将会变得非常大。

因此， DNS 服务必须设置好高可用，高并发与分布式。



- Root DNS server: Returns the IP address of the top level domain DNS servers
- Top level domain DNS server: Returns the IP address of the authoritative DNS server
- Authoritative DNS server: Returns the IP address of the cooresponding host

- 根域名服务器：返回顶级域名的 DNS 服务的 IP 地址
- 顶级域名服务器: 返回权威 DNS 服务的 IP 地址
- 权威 DNS 服务器：返回对应主机的 IP 地址


DNS Resolution Process
To improve DNS resolution performance, many networks deploy DNS cache servers based on location. The DNS resolution process looks like:


## DNS 解析过程 

为了提升 DNS 解析性能，众多网络基于地理位置部署了 DNS 缓存服务。

DNS 解析过程看起来如下：

- The client computer will send out a DNS request, asking what IP of “google.com” is, and it will look for the IP address of “google.com” in the browser cache first.
- 客户端电脑会发起一个 DNS 请求，询问 `google.com` 对应的 IP 是什么，并且它会从浏览器本地缓存优先查询


- Then the request will be sent to local DNS server. The local DNS server is automatically assigned by your internet service provider (ISP), usually it is the router that provided by your ISP.
- 然后请求将被发送到本地 DNS 服务器。 本地 DNS 服务器由您的 Internet 服务提供商 (ISP) 自动分配，通常是您的 ISP 提供的路由器。


- The local DNS server receives the DNS request from client and it will look for the IP address of “google.com” in its cache. If it can find a corresponding entry, it will return the IP address directly to client. If not, the local DNS server will ask its root domain name server: “Can you tell me the IP address of google.com”? The root domain name server is the highest level, and there are 13 sets in the world. It is not directly used for domain name resolution, but can point a way.
- 本地 DNS 服务从客户端接收到 DNS 请求，它会从当前缓存查询`google.com` 的 IP 地址。如果能够找到则返回 IP 地址给客户端。否则，本地 DNS 服务会询问根服务器：“能够告诉我 `google.com` 的 IP 吗？”。根域名服务是最高级别服务，在全世界范围内有 13 台。它并不直接用于域名解析，而是指出一条方向道路。

- The root DNS server receives the request from local DNS, finds that the suffix is .com , then it tells the local DNS “okay, you are looking for google.com, since it is a .com suffix, it is managed by the .com zone, here is the IP address of the top level domain server for .com zone, go for it”.
- 根 DNS 服务从本地 DNS 服务接收到请求，发现这是一个后缀为 .com 的域名，它会告知本地 DNS 服务：“好吧伙计，你现在找的 `google.com` 因为它是 .com 后缀，由 .com 区域管理的，这里是 .com 域名顶级域名服务，去吧”

- The local DNS server turns to the .com zone top level domain namer server and ask for the IP address of “google.com”. The top-level .com zone DNS server again, points a direction for this request. It provides the IP address of the authoritative DNS server that responsible for google.com .
- 本地 DNS 服务转向 .com 区域顶级域名服务寻找 google.com 的 IP。顶级域名 .com 区域 DNS 服务同样为客户端的请求指出一个方向。它提供了为 google.com 提供认证 DNS 服务的 IP

- The local DNS server then turns to the authoritative DNS server and ask for the IP address of “google.com”. This time the authoritative DNS server of google.com is the original source of the domain name resolution result. It returns the IP address of google.com to the local DNS server.
- 本地 DNS 服务转向 DNS 认证服务寻找 google.com 的 IP。