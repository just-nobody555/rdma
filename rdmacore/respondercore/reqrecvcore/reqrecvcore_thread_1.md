# ReqRecvCore\_Thread\_1

## 模块功能

ReqRecvCore\_Thread\_1 是 RDMA 接收路径中 ReqRecvCore 模块的第一阶段处理单元，其核心功能包括：

* 接收来自 MAC 解封装模块的入向数据包头部；\
  从包头中提取目标 QP 号（Remote QPN）；
* 构造上下文（Cxt）读取请求，包含QPN、操作类型（CXT\_READ）和公共头信息，向 OoOStation 发起 QP 上下文（Context）读取请求；
* 将原始包头作为请求数据传递给后续阶段，为权限验证、WQE 匹配和 DMA 执行提供网络元数据（Net Meta）。

## 模块接口

<table><thead><tr><th width="191">接口</th><th width="95">输入/输出</th><th width="87">位宽</th><th width="87">对接模块</th><th width="247">说明</th></tr></thead><tbody><tr><td>clk</td><td>input</td><td>1</td><td>全局时钟</td><td>系统主时钟</td></tr><tr><td>rst</td><td>input</td><td>1</td><td>全局复位</td><td>高有效异步复位</td></tr><tr><td>ingress_pkt_valid</td><td>input</td><td>1</td><td>MACDecap</td><td>入向包有效信号</td></tr><tr><td>ingress_pkt_head</td><td>input</td><td>488</td><td>MACDecap</td><td>包头元数据（含 opcode、QPN、地址等）</td></tr><tr><td>ingress_pkt_ready</td><td>output</td><td>1</td><td>MACDecap</td><td>本模块接收就绪信号（流控）</td></tr><tr><td>fetch_cxt_ingress_valid</td><td>output</td><td>1</td><td>OoOStation</td><td>上下文请求有效信号</td></tr><tr><td>fetch_cxt_ingress_head</td><td>output</td><td>160</td><td>OoOStation</td><td>上下文请求头（含 QPN、opcode、通用头）</td></tr><tr><td>fetch_cxt_ingress_data</td><td>output</td><td>488</td><td>OoOStation</td><td>上下文请求数据（原始包头）</td></tr><tr><td>fetch_cxt_ingress_start</td><td>output</td><td>1</td><td>OoOStation</td><td>请求事务起始标志（单拍置1）</td></tr><tr><td>fetch_cxt_ingress_last</td><td>output</td><td>1</td><td>OoOStation</td><td>请求事务结束标志（单拍置1）</td></tr><tr><td>fetch_cxt_ingress_ready</td><td>input</td><td>1</td><td>OoOStation</td><td>OoOStation 接收就绪信号（流控）</td></tr></tbody></table>

## 状态机设计

### 状态定义

<table><thead><tr><th width="95">状态名</th><th width="111">编码（3'b）</th><th width="405">说明</th></tr></thead><tbody><tr><td>IDLE_s</td><td>3'b001</td><td>空闲状态：等待入向包到达</td></tr><tr><td>JUDGE_s</td><td>3'b010</td><td>判决状态：准备发起上下文请求</td></tr><tr><td>FETCH_CXT_s</td><td>3'b011</td><td>获取上下文状态：向 OoOStation 发送请求</td></tr></tbody></table>

### 状态转移表

<table><thead><tr><th width="95">现态</th><th width="95">次态</th><th width="247">转移条件</th><th width="247">中文说明</th></tr></thead><tbody><tr><td>IDLE_s</td><td>JUDGE_s</td><td>ingress_pkt_valid == 1</td><td>有有效入向包到达，进入判决状态</td></tr><tr><td>IDLE_s</td><td>IDLE_s</td><td>ingress_pkt_valid == 0</td><td>无有效包，保持空闲</td></tr><tr><td>JUDGE_s</td><td>FETCH_CXT_s</td><td>1</td><td>无条件跳转，进入上下文请求状态</td></tr><tr><td>FETCH_CXT_s</td><td>IDLE_s</td><td>fetch_cxt_ingress_valid == 1 &#x26;&#x26; fetch_cxt_ingress_ready == 1</td><td>请求被OoOStation接收，返回空闲</td></tr><tr><td>FETCH_CXT_s</td><td>FETCH_CXT_s</td><td>fetch_cxt_ingress_valid == 0 || fetch_cxt_ingress_ready == 0</td><td>握手未完成，继续等待</td></tr><tr><td>default</td><td>IDLE_s</td><td>无条件</td><td>异常状态，安全回退到空闲</td></tr></tbody></table>

上电或复位后，首先进入 IDLE\_s 状态。

* 在 IDLE\_s 状态，模块持续监测输入信号 ingress\_pkt\_valid：
  * 如果 ingress\_pkt\_valid == 1，表示有有效入向包到达，则立即转移到 JUDGE\_s 状态；
  * 否则保持在 IDLE\_s，并持续拉高 ingress\_pkt\_ready 信号，表示可接收新包。

进入 JUDGE\_s 状态后，模块不进行任何条件判断或数据处理，而是无条件地、立即转移到 FETCH\_CXT\_s 状态。此状态的存在主要是为了逻辑分层和未来扩展，当前仅为一个过渡跳转。

在 FETCH\_CXT\_s 状态，模块执行以下操作：

* 拉高 fetch\_cxt\_ingress\_valid 信号，向 OoOStation 发起 QP 上下文请求；
* 持续监测 OoOStation 的流控响应信号 fetch\_cxt\_ingress\_ready；
* 若 fetch\_cxt\_ingress\_valid == 1 且 fetch\_cxt\_ingress\_ready == 1，表示握手成功，则认为上下文请求已完成，返回 IDLE\_s 状态，准备处理下一个包；
* 否则，OoOStation 尚未准备好，模块保持在 FETCH\_CXT\_s 状态，继续等待，直到握手成功。
