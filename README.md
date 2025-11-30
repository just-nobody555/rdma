# RDMAcore

RDMA Core负责与RDMA通信相关的协议处理。

RDMA Core中包含两个大核，而每个大核内又分为两个小核。

按照处理本节点发起的操作和处理远端节点发起的操作，RDMA Core由Requester Core和Responder Core组成。Requester Core负责处理本机主动发起的操作，其中Requester Transport Core承接并处理主机下发的通信请求；而Response Receive Core负责接收并处理远端主机接收到本机传输的请求后返回的ACK响应。与之相对应的是Responder Core负责处理远端主机主动发起的操作，其中Requester Receive Core负责接收并处理需要从网络向主存中写入数据的请求，例如Send和Write请求；而Response Transport Core负责接收并处理需要从主存中读取数据并发送到网络链路上的请求，例如Read请求。

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

<br>
