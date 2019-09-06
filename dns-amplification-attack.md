# DNS反射/放大攻击

DDOS攻击的一种，利用DNS回复包比请求包大（生僻或者不存在的域名，迭代查询），放大流量，伪造源IP（UDP），把流量打到受害者服务器（流量来自正常的DNS服务器，查不到攻击者IP）。

反射，放大其实是说的两方面
+ 反射，是伪造源IP，把回复打到受害方；
+ 放大，是返回的消息大小大于请求的消息，放大倍数越大，攻击效果越好。

解决
+ 直接关闭DNS
+ 只响应安全域内的请求
+ 限制DNS包的大小

## DNS

13个不同IP地址（用Anycast，不是13台，同IP不同机器）的根域名服务器（A - M），使用13个是为了保证信息能存储在一个最小UDP包内（512字节）。

## 查询过程

本地向本地域名服务器递归查询：发起请求，等结果就好了

本地域名服务器进行迭代查询（可能有Cache）：向根域名服务器查询，然后查顶级域名服务器，逐级查询。要与多个服务器进行通信，同时只收发一条消息，用UDP消耗更低。

## Reference

+ [Difference between Amplification and Reflection Attack?
](https://security.stackexchange.com/questions/181121/difference-between-amplification-and-reflection-attack)
+ [为什么DNS适合使用UDP协议而不是TCP协议？](http://www.xumenger.com/dns-udp-tcp-20180604/)
+ [浅析DNS反射放大攻击](https://cloud.tencent.com/developer/article/1463080)
