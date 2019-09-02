## 为什么本地需要存储子网掩码

对于点对点通信来说，对于本地机器来说，发送只有两个方向，一个是发到本地，一个是发到远端，只需要根据IP地址区分即可，不需要子网掩码。

在以太网中，为什么除了路由之外，本地也要存储子网掩码？局域网中，发送方向有三个，一个是本地，另外两个分别是子网内和子网外。发送本地依然可以根据IP直接判断；如果是发送子网内（根据子网掩码判断），则通过ARP协议获取目标的MAC地址发送；如果发送的是子网外，则发送到网关（通过ARP获取网关MAC地址）。简单来说，以太网是通过MAC地址做单播，在本地发送时必须要通过子网掩码判断发送的是子网内的目标还是需要发送到网关。在网关的数据链路层只看到MAC地址，没有IP信息，如果MAC地址不是路由，就直接被丢弃了。


## 为什么需要MAC地址和IP地址

MAC地址可以认为是一个物理概念，一块网卡对应一个地址，网卡可以置于不同的子网内，所以根据MAC地址是无法区分是否处于同一个子网。

MAC地址也只能用于同一个子网内收发，因为MAC地址有48位，做不了聚合。假如没有最终目标的IP地址，只有MAC地址，路由将要建立很庞大的表来确定通过哪个端口转发。所有IP地址用于区分子网内还是子网外，并且用于路由聚合。

因此有了MAC地址后，还需要有IP地址用于跨子网的访问。

反过来，有IP地址为什么还需要MAC地址，因为TCP/IP协议设计的时候就没有考虑说要统治全球，当时除了以太网外，还有ATM、令牌环网等。最后以太网获得胜利，就是目前的TCP/IP over Ethernet，MAC地址和IP地址同时存在。


## SSL/TLS

### 命名

SSL 3.0之后升级为TLS 1.0。

TLS 1.0也被标示为SSL 3.1，TLS 1.1为SSL 3.2，TLS 1.2为SSL 3.3。

### 流程

#### ClientHello

+ 客户端发送支持的协议版本（TLS）、加密方式（RSA）和压缩方式
+ 发送客户端随机数A
+ 发送信息不包括域名，因此一个IP只能有一个证书。TLS加入了Server Name Indication扩展，客户端可以提供要请求的用户，方便虚拟主机用户。

#### ServerHellp

+ 确认协议版本（TLS）、加密方式
+ 发送服务端随机数B
+ 发送服务器证书

#### 客户端回应

证书校验（包括有效期），证书必定是由上一级证书签发（私钥加密），客户端本地有一系列可信任证书，可通过可信任证书（公钥）确认证书是否合法。

+ 回复客户端随机数（用公钥加密，pre-master key）
+ 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送
+ 客户端握手结束通知，前面发送的所有内容的hash值

三个随机数用于生成对称加密的秘钥。用三个是因为避免客户端生成的随机数质量差，三个伪随机数十分接近随机。

#### 服务器最后回应

+ 编码改变通知，表示随后的信息都将用双方商定的加密方法和密钥发送
+ 服务器握手结束通知，同时也是前面发送的所有内容的hash值


TCP为什么需要建立连接

确认第一个序号


## Reference

[SSL/TLS协议运行机制的概述](https://www.ruanyifeng.com/blog/2014/02/ssl_tls.html)