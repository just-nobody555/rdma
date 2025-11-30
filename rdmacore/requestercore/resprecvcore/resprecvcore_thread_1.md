# RespRecvCore\_Thread\_1

## 模块功能

* 从 PacketDeparser 送来的已解析网络响应包中提取本端 QP 号
* 向 OoOStation 发起“查询该 QP 上下文（Context）”的请求
* 将原始包头作为附带数据一并传递，为后续 WQE 查询、CQE 生成、DMA 写提供元数据支持。

## 模块接口

<table><thead><tr><th width="191">信号名称</th><th width="87">方向</th><th width="87">位宽</th><th width="119">对接模块</th><th width="247">说明</th></tr></thead><tbody><tr><td>clk</td><td>input</td><td>1</td><td>全局时钟域</td><td>上升沿驱动</td></tr><tr><td>rst</td><td>input</td><td>1</td><td>全局复位</td><td>同步高有效复位</td></tr><tr><td>ingress_pkt_valid</td><td>input</td><td>1</td><td>PacketDeparser</td><td>包有效信号</td></tr><tr><td>ingress_pkt_head</td><td>input</td><td>488</td><td>PacketDeparser</td><td>包元数据，含 QP 号、opcode、地址等</td></tr><tr><td>ingress_pkt_ready</td><td>output</td><td>1</td><td>PacketDeparser</td><td>本模块就绪信号，仅在 IDLE 态为高</td></tr><tr><td>fetch_cxt_ingress_valid</td><td>output</td><td>1</td><td>OoOStation</td><td>CXT 查询请求有效</td></tr><tr><td>fetch_cxt_ingress_head</td><td>output</td><td>160</td><td>OoOStation</td><td>CXT 请求头</td></tr><tr><td>fetch_cxt_ingress_data</td><td>output</td><td>256</td><td>OoOStation</td><td>附带数据 </td></tr><tr><td>fetch_cxt_ingress_start</td><td>output</td><td>1</td><td>OoOStation</td><td>分段起始标志（本设计恒为 1）</td></tr><tr><td>fetch_cxt_ingress_last</td><td>output</td><td>1</td><td>OoOStation</td><td>分段结束标志（本设计恒为 1）</td></tr><tr><td>fetch_cxt_ingress_ready</td><td>input</td><td>1</td><td>OoOStation</td><td>OoOStation 就绪信号</td></tr></tbody></table>

## 状态机设计

此模块状态设计较为简单，模块复位后处于 IDLE 状态，等待输入包。当有包到来（ingress\_pkt\_valid 有效）时，进入 JUDGE 状态，紧接着进入 FETCH\_CXT 状态。在 FETCH\_CXT 状态，模块发出 CXT 查询请求，并一直保持，直到 OoOStation 返回 ready 信号。握手成功后，模块回到 IDLE 状态，准备处理下一个包。整个过程一次只处理一个包，确保顺序和可靠性。

### 状态定义

<table><thead><tr><th width="95">状态名</th><th width="87">编码</th><th width="396.142822265625">说明</th></tr></thead><tbody><tr><td>IDLE_s</td><td>3'b001</td><td>空闲态：等待新包到来</td></tr><tr><td>JUDGE_s</td><td>3'b010</td><td>判定态：极短过渡态（无实际逻辑）</td></tr><tr><td>FETCH_CXT_s</td><td>3'b011</td><td>请求态：持续发送 CXT 请求，直到被接收</td></tr></tbody></table>

### 状态转移表

<table><thead><tr><th width="175">现态</th><th width="151">次态 </th><th width="230.4283447265625">转移条件 </th><th width="247">说明</th></tr></thead><tbody><tr><td>IDLE_s</td><td>JUDGE_s</td><td>ingress_pkt_valid == 1</td><td>有新包到达，进入判定</td></tr><tr><td>IDLE_s</td><td>IDLE_s</td><td>ingress_pkt_valid == 0</td><td>无包，保持空闲</td></tr><tr><td>JUDGE_s</td><td>FETCH_CXT_s</td><td>无条件</td><td>极短过渡，直接进入请求态</td></tr><tr><td>FETCH_CXT_s</td><td>IDLE_s</td><td>fetch_cxt_ingress_valid &#x26;&#x26; fetch_cxt_ingress_ready</td><td>请求被 OoOStation 接收，返回空闲</td></tr><tr><td>FETCH_CXT_s</td><td>FETCH_CXT_s</td><td>!(fetch_cxt_ingress_valid &#x26;&#x26; fetch_cxt_ingress_ready)</td><td>请求未被接收，持续尝试</td></tr></tbody></table>
