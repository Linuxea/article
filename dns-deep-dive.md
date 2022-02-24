# DNS Deep Dive


## DNS Server

In real world, when you watching news, making online orders, downloading files or listening to live broadcasting, you need to visit the website by domain names, for example, youtube.com, google.com, ..etc. You must also remember the name of these website, not their IP addresses, because IP addresses are very difficult to remember, compare to website names. Therefore you need an website/domain name address book, which the the DNS server.


在现实世界上网冲浪，当你观看新闻，购物，下载文件或者听广播时，你需要通过域名来访问网站，比如 `youtube.com`, `google.com` 等等。

你必须记住它们的网址而不是 `IP`，因为相比下 IP 地址并不容易被记住。

因此你需要一个域名地址簿，也就 `DNS` 服务器。



DNS is very important in everyday life. Everyone online needs to access it, but at the same time, this is also a very big challenge for it. If the DNS server fails, the entire internet will be down.



DNS 在日常生活非常重要。

每一次上网冲浪都需要对其访问，但同时这也是一个巨大的挑战。

如果 DNS 服务宕机，整个互联网世界都会下线。

In addition, people who surf the Internet are distributed all over the world, if everyone goes to the same place to access a certain server, the delay will be very large. Therefore, the DNS server must be set to be highly available, highly concurrent and distributed.

另外，上网冲浪的人分布世界各地，如果每个人访问同个地点的一个中心服务，延迟将会变得非常大。

因此，DNS 服务必须设置高可用，高并发与分布式。



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
- 客户端电脑会发起一个 DNS 请求，询问 `google.com` 对应的 `IP` 是什么，并且它会从浏览器本地缓存优先查询


- Then the request will be sent to local DNS server. The local DNS server is automatically assigned by your internet service provider (ISP), usually it is the router that provided by your ISP.
- 然后请求将被发送到本地 DNS 服务器。 本地 DNS 服务器由您的 Internet 服务提供商 (ISP) 自动分配，通常是您的 ISP 提供的路由器。


- The local DNS server receives the DNS request from client and it will look for the IP address of “google.com” in its cache. If it can find a corresponding entry, it will return the IP address directly to client. If not, the local DNS server will ask its root domain name server: “Can you tell me the IP address of google.com”? The root domain name server is the highest level, and there are 13 sets in the world. It is not directly used for domain name resolution, but can point a way.
- 本地 DNS 服务从客户端接收到 DNS 请求，它会从当前缓存查询`google.com` 的 IP 地址。如果能够找到则返回 IP 地址给客户端。否则，本地 DNS 服务会询问根服务器：“能够告诉我 `google.com` 的 IP 吗？”。根域名服务是最高级别服务，在全世界范围内有 13 台。它并不直接用于域名解析，而是指出一条方向道路。

- The root DNS server receives the request from local DNS, finds that the suffix is .com , then it tells the local DNS “okay, you are looking for google.com, since it is a .com suffix, it is managed by the .com zone, here is the IP address of the top level domain server for .com zone, go for it”.
- 根 DNS 服务从本地 DNS 服务接收到请求，发现这是一个后缀为 `.com` 的域名，它会告知本地 DNS 服务：“好吧伙计，你现在找的 `google.com` 因为它是 .com 后缀，由 .com 区域管理的，这里是 .com 域名顶级域名服务，去吧”

- The local DNS server turns to the .com zone top level domain namer server and ask for the IP address of “google.com”. The top-level .com zone DNS server again, points a direction for this request. It provides the IP address of the authoritative DNS server that responsible for google.com .
- 本地 DNS 服务转向 .com 区域顶级域名服务寻找 google.com 的 IP。顶级域名 .com 区域 DNS 服务同样为客户端的请求指出一个方向。它提供了为 google.com 提供认证 DNS 服务的 IP

- The local DNS server then turns to the authoritative DNS server and ask for the IP address of “google.com”. This time the authoritative DNS server of google.com is the original source of the domain name resolution result. It returns the IP address of google.com to the local DNS server.
- 本地 DNS 服务转向 DNS 权威服务寻找 google.com 的 IP。这次 google.com 的权威 DNS 服务器是域名解析结果的原始来源。它向本地 DNS 服务返回了 google.com 的 IP 地址。




## DNS Load Balancing

Internal Load Balancing
### Internal Load Balancing
DNS server does internal load balancing first. For example, if an application wants to access the database, should the IP address of the database be configured in the application, or should the domain name of the database be configured?

DNS 服务首先在使用了内部负载均衡。
比如，如果应用需要访问数据库，是因为在应用中配置数据库的 IP 地址，还是应该配置数据库的域名？

Obviously, the domain name should be configured, because once the database is changed to another machine for some reason, and if multiple applications are configured with this database, once the IP address is changed, all these applications need to be modified again.


很明显，应该配置的是域名。一旦数据库因为某些原因迁移到另外的机器，配置此数据库的应用都需要重新修改。

However, if the domain name is configured, as long as the domain name is mapped to a new IP address in the DNS server, this work is completed, which greatly simplifies the operation and maintenance.

然而，如果配置的是数据库域名地址，只要在 DNS 服务将域名映射为新的 IP 地址，工作就完成了，这简化了操作与维护的工作。

On this basis, we can go further. For example, how to load balance among multiple applications accessing it? Just configure it as a domain name. In the domain name resolution, we only need to configure the policy, this time return the first IP, next time return the second IP, we can achieve load balancing.

在这个基础上，我们可以更进一步。


比如，如何在多应用访问数据时进行负载均衡？

只需要通过配置为域名地址。在域名解析过程中，我们只需要配置策略，这一次返回的是第一个 IP，下一次返回第二个 IP ... ... 就可以实现负载均衡。


### Global Load Balancing

In order to ensure high availability of our applications, they are often deployed in multiple computer data centers, and each place will have its own IP address.

为了确保应用的高可用，需要把它们部署到多个计算机数据中心，并且每个中心都有自己的 IP 地址。


When a user accesses a domain name, this IP address can poll multiple data centers. If a data center is down for some reason, as long as the IP address corresponding to the data center is deleted in the DNS server, a certain degree of high availability can be achieved.

当用户访问一个域名时，这个IP地址可以轮询多个数据中心。 如果某个数据中心由于某种原因宕机了，只要在DNS服务器中删除该数据中心对应的IP地址，就可以实现一定程度的高可用。


In addition, we definitely want users in New York to visit the data center in New York, and users in Seattle to visit the data center in Seattle, so that the customer experience will be very good and the access speed will be super fast. This is the concept of global load balancing.

除此之外，我们明确需要纽约的用户访问处于纽约的数据中心，西雅图的用户访问处于西雅图的数据中心，这样用户体验会非常好，访问速度也会非常快。
这就是全局负载均衡概念。


Let’s take a look at how this works, suppose there are multiple regions across the country, and each region has three Available Zones.

让我们看下全局负载是如何工作的。

假设全国有多个 `region`，每个地区拥有三个可用的 `Zone`.


- When a client wants to access app.metaleap.com, it needs to convert the domain name to IP address for access, so it needs to request the local DNS resolver.
- 当一个客户端需要访问 app.metaleap.com，它需要将域名转换成 IP 来访问，所以它需要请求本地 DNS 解析 


- The local DNS resolver first checks to see if the local cache has this record. If there is, use it directly.
- 本地 DNS 解析首先检测本地缓存是否存在该记录，如果存在，则直接使用。

- If there is no local cache, you need to request the local DNS server.
- 如果不存在本地缓存，需要请求本地 DNS 服务


- The local DNS server also needs to check whether there is a cache locally, and if so, return it.
- 本地 DNS 服务同样需要检查是否存在本地缓存，如果有则返回。

- If there is no local DNS, the local DNS needs to recursively find the top-level domain name server of .com from the root DNS server, and finally find the authoritative DNS server of metaleap.com, and give it to the local DNS server. The authoritative DNS server will normally return true IP address to access.
- 如果没有本地 DNS 记录，本地 DNS 服务需要递归地从根 DNS 服务找到顶级域名服务，最终找到 metaleap.com 权威 DNS 服务，权威 DNS 服务返回真实的 IP 地址。 



For simple applications that do not require global load balancing, the authoritative DNS server of metaleap.com can directly resolve the domain name app.metaleap.com to one or more IP addresses, and then the client can use multiple IP addresses to simple polling for simple load balancing.

对于简单的应用来说，它们不需要全局负载均衡，认证 DNS 服务可以直接解析域名 app.metaleap.com 为一个或多个 IP 地址。
客户端可以使用多 IP 地址轮询来实现简单负载均衡。


However, for complex applications, especially large-scale applications that cross regions and data centers, a more complex global load balancing mechanism is required, so a dedicated device or server is required to do this. This is the global load balancer (GSLB , Global Load Balancer).


然而，对于复杂应用来讲，特别是跨 region 与数据中心的大规模应用，对于一个更加复杂的全局负载均衡的需求是必要的，因此需要`专用设备`或`服务器`来执行此操作。
这就是`全局负载均衡`（GSLB, Global Load Balancer）。



In the DNS server of metaleap.com, generally by configuring CNAME, an alias is given to app.metaleap.com, such as app.vip.metaleap.com, and then tell the local DNS server to request GSLB to resolve the domain name, GSLB can achieve load balancing through its own strategy in the process of resolving this domain name.


在 metaleap.com 的 DNS 服务器中，一般通过配置 CNAME，给 app.metaleap.com 起一个别名，比如 app.vip.metaleap.com ，然后告诉本地 DNS 服务器请求 GSLB 解析域名 ，GSLB 在解析这个域名的过程中，可以通过自己的策略实现负载均衡。


The two-layer GSLB is drawn in the diagram because it is divided into data centers and regions. We hope that customers of different data centers can access resources in the same data center to improving throughput and reducing latency.

图中绘制了两层GSLB，因为它分为数据中心和区域。 我们希望不同数据中心的客户可以访问同一个数据中心的资源，以提高吞吐量并减少延迟。 


Conclusion
DNS is the address book of the network world, and addresses can be searched through domain names, because domain name servers are organized in a tree structure, so domain name search uses a recursive method and enhances performance through caching.
In the process of mapping between domain names and IPs, applications are given the opportunity to perform load balancing based on domain names, which can be simple load balancing or global load balancing based on addresses, data center and region.


## Conclusion

DNS is the address book of the network world, and addresses can be searched through domain names, because domain name servers are organized in a tree structure, so domain name search uses a recursive method and enhances performance through caching.

DNS 是网络世界的地址簿，地址是可以通过域名来找到的，因为域名解析的服务器是以树状结构组织，所以搜索以递归的方式并且通过缓存来加强性能。


In the process of mapping between domain names and IPs, applications are given the opportunity to perform load balancing based on domain names, which can be simple load balancing or global load balancing based on addresses, data center and region.

在域名与 IP 的映射处理过程中，应用被赋予了基于域名来执行负载均衡的机会，这可以是基于地址，数据中心，区域的简单负载或者复杂负载。



## 参考

- [1] [DNS Deep Dive](https://medium.com/geekculture/dns-deep-dive-421e321a0a06)
- [2] [DNS全局负载均衡（GSLB）基本原理](https://www.cnblogs.com/foxgab/p/6900101.html)
- [3] [域名系统](https://baike.baidu.com/item/%E5%9F%9F%E5%90%8D%E7%B3%BB%E7%BB%9F/2251573?fromtitle=DNS&fromid=427444&fr=aladdin)