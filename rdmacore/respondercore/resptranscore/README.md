# RespTransCore

## 模块功能

RespTransCore 负责生成RDMA网络响应数据包，其中包含一个RespTransCore\_Thread\_1，主要功能包括：

* 网络响应生成
  * 从请求接收核心（ReqRecvCore）获取响应元数据（通过 net\_resp\_dout）。
  * 从数据收集模块（Gather Data）获取有效载荷数据（512-bit 的 payload\_data）。
  * 将元数据与有效载荷组合，构建成完整的网络响应数据包。
* 有效载荷缓冲管理
  * 通过 Payload Buffer 存储有效载荷数据：
    * 发送写入请求将 payload\_data 存入缓冲区。
    * 接收写入确认，用于后续数据包组装。
* 数据包输出
  * 将组装好的响应数据包通过 传输子系统（TransportSubsystem） 发送：
    * 通过 数据包元数据输出数据包。

## 模块接口

<table><thead><tr><th width="143">信号名</th><th width="95">输入/输出</th><th width="87">位宽</th><th width="151">对接模块</th><th width="247">中文说明</th></tr></thead><tbody><tr><td>clk</td><td>input</td><td>1</td><td>无（全局时钟）</td><td>时钟信号</td></tr><tr><td>rst</td><td>input</td><td>1</td><td>无（全局复位）</td><td>复位信号（高有效）</td></tr><tr><td>net_resp_ren</td><td>output</td><td>1</td><td>ReqRecvCore</td><td>从ReqRecvCore读取响应数据的使能信号</td></tr><tr><td>net_resp_empty</td><td>input</td><td>1</td><td>ReqRecvCore</td><td>来自ReqRecvCore的响应队列空标志</td></tr><tr><td>net_resp_dout</td><td>input</td><td>512</td><td>ReqRecvCore</td><td>来自ReqRecvCore的响应元数据</td></tr><tr><td>payload_empty</td><td>input</td><td>1</td><td>Gather Data</td><td>有效载荷数据缓冲区空标志</td></tr><tr><td>payload_data</td><td>input</td><td>512</td><td>Gather Data</td><td>512位有效载荷数据</td></tr><tr><td>payload_ren</td><td>output</td><td>1</td><td>Gather Data</td><td>读取有效载荷数据的使能信号</td></tr><tr><td>insert_req_valid</td><td>output</td><td>1</td><td>Payload Buffer</td><td>向Payload Buffer插入数据的请求有效信号</td></tr><tr><td>insert_req_start</td><td>output</td><td>1</td><td>Payload Buffer</td><td>插入请求的起始段标志</td></tr><tr><td>insert_req_last</td><td>output</td><td>1</td><td>Payload Buffer</td><td>插入请求的结束段标志</td></tr><tr><td>insert_req_head</td><td>output</td><td>9</td><td>Payload Buffer</td><td>插入请求的头部信息（总段数）</td></tr><tr><td>insert_req_data</td><td>output</td><td>512</td><td>Payload Buffer</td><td>插入请求的512位数据段内容</td></tr><tr><td>insert_req_ready</td><td>input</td><td>1</td><td>Payload Buffer</td><td>Payload Buffer接收插入请求的就绪信号</td></tr><tr><td>insert_resp_valid</td><td>input</td><td>1</td><td>Payload Buffer</td><td>Payload Buffer插入响应的有效标志</td></tr><tr><td>insert_resp_data</td><td>input</td><td>9</td><td>Payload Buffer</td><td>Payload Buffer返回的插入起始地址</td></tr><tr><td>egress_pkt_valid</td><td>output</td><td>1</td><td>TransportSubsystem</td><td>输出数据包有效标志</td></tr><tr><td>egress_pkt_head</td><td>output</td><td>488</td><td>TransportSubsystem</td><td>输出数据包的元数据头部</td></tr><tr><td>egress_pkt_ready</td><td>input</td><td>1</td><td>TransportSubsystem</td><td>TransportSubsystem接收数据包的就绪信号</td></tr></tbody></table>
