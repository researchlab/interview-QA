##### http keep-alive
普通的http连接是客户端连接上服务端，然后结束请求后，由客户端或者服务端进行http连接的关闭。下次再发送请求的时候，客户端再发起一个连接，传送数据，关闭连接。这么个流程反复。但是一旦客户端发送connection:keep-alive头给服务端，且服务端也接受这个keep-alive的话，两边对上暗号，这个连接就可以复用了，一个http处理完之后，另外一个http数据直接从这个连接走了。减少新建和断开TCP连接的消耗。这个可以在Nginx设置，
```sh
keepalive_timeout   65;
```
这个keepalive_timout时间值意味着：一个http产生的tcp连接在传送完最后一个响应后，还需要hold住keepalive_timeout秒后，才开始关闭这个连接。

特别注意TCP层的keep alive和http不是一个意思。TCP的是指：tcp连接建立后，如果客户端很长一段时间不发送消息，当连接很久没有收到报文，tcp会主动发送一个为空的报文（侦测包）给对方，如果对方收到了并且回复了，证明对方还在。如果对方没有报文返回，重试多次之后则确认连接丢失，断开连接。

tcp的keep alive可通过

```sh
sysctl -a|grep tcp_keepalive
#打印
net.ipv4.tcp_keepalive_intvl = 75
net.ipv4.tcp_keepalive_probes = 9
net.ipv4.tcp_keepalive_time = 7200 
```
net.ipv4.tcp_keepalive_intvl = 75 // 当探测没有确认时，重新发送探测的频度。缺省是75秒。

net.ipv4.tcp_keepalive_probes = 9 //在认定连接失效之前，发送多少个TCP的keepalive探测包。缺省值是9。这个值乘以tcp_keepalive_intvl之后决定了，一个连接发送了keepalive之后可以有多少时间没有回应

net.ipv4.tcp_keepalive_time = 7200 //当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时。一般设置为30分钟1800

修改：
```sh
vi /etc/sysctl.conf
## 键入
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_time = 1800
## 生效
sysctl -p
```

##### http能不能一次连接多次请求，不等后端返回

可以

##### tcp与udp区别，udp优点，适用场景

tcp是面向连接的，upd是无连接状态的。

udp相比tcp没有建立连接的过程，所以更快，同时也更安全，不容易被攻击。upd没有阻塞控制，因此出现网络阻塞不会使源主机的发送效率降低。upd支持一对多，多对多等，tcp是点对点传输。tcp首部开销20字节，udp8字节。

udp使用场景：视频通话、im聊天等。

###### time-wait的作用
time-wait表示客户端等待服务端返回关闭信息的状态，closed_wait表示服务端得知客户端想要关闭连接，进入半关闭状态并返回一段TCP报文。

time-wait作用：

防止上一次连接的包迷路后重复出现，影响新连接
可靠的关闭TCP连接。在主动关闭方发送的最后一个 ack(fin) ，有可能丢失，这时被动方会重新发fin, 如果这时主动方处于 CLOSED 状态 ，就会响应 rst 而不是 ack。所以主动方要处于 TIME_WAIT 状态，而不能是 CLOSED 。另外这么设计TIME_WAIT 会定时的回收资源，并不会占用很大资源的，除非短时间内接受大量请求或者受到攻击。
解决办法：
```sh
#对于一个新建连接，内核要发送多少个 SYN 连接请求才决定放弃,不应该大于255，默认值是5，对应于180秒左右时间

net.ipv4.tcp_syn_retries=2

#net.ipv4.tcp_synack_retries=2

#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为300秒

net.ipv4.tcp_keepalive_time=1200

net.ipv4.tcp_orphan_retries=3

#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间

net.ipv4.tcp_fin_timeout=30  

#表示SYN队列的长度，默认为1024，加大队列长度为8192，可以容纳更多等待连接的网络连接数。

net.ipv4.tcp_max_syn_backlog = 4096

#表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭

net.ipv4.tcp_syncookies = 1

#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭

net.ipv4.tcp_tw_reuse = 1

#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭

net.ipv4.tcp_tw_recycle = 1
 
##减少超时前的探测次数 

net.ipv4.tcp_keepalive_probes=5 

##优化网络设备接收队列 

net.core.netdev_max_backlog=3000 
```
close_wait:
被动关闭，通常是由于客户端忘记关闭tcp连接导致。

https://blog.csdn.net/yucdsn/article/details/81092679

