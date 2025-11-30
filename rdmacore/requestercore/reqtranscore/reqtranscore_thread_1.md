# ReqTransCore\_Thread\_1

## 模块功能

接收来自 WQEParser 的WQE；在检测到有效 WQE 后，锁存该 WQE 元数据；从中提取 QPN 等关键字段；构造发往 OoOStation 的含请求头与原始 WQE 数据上下文读取请求；向 OoOStation 乱序站发起一次上下文读取请求；通过 valid/ready 握手机制 与上下游模块进行流控，确保数据可靠传输；每次仅处理一个 WQE，处理完成后才接受下一个。

## 模块接口

<table><thead><tr><th width="207">信号名称</th><th width="87">方向</th><th width="247">位宽</th><th width="107">对接模块</th><th width="247">说明</th></tr></thead><tbody><tr><td>`clk`</td><td>input</td><td>1</td><td>全局时钟域</td><td>上升沿采样，驱动所有时序逻辑</td></tr><tr><td>`rst`</td><td>input</td><td>1</td><td>全局复位</td><td>高电平有效，同步复位整个模块</td></tr><tr><td>`sub_wqe_valid`</td><td>input</td><td>1</td><td>WQEParser</td><td>表示 `sub_wqe_meta` 当前有效</td></tr><tr><td>`sub_wqe_meta`</td><td>input</td><td>`WQE_META_WIDTH`</td><td>WQEParser</td><td>WQE 元数据，包含 QPN 等字段</td></tr><tr><td>`sub_wqe_ready`</td><td>output</td><td>1</td><td>WQEParser</td><td>本模块准备好接收新 WQE（流控）</td></tr><tr><td>`fetch_cxt_ingress_valid`</td><td>output</td><td>1</td><td>OoOStation</td><td>本模块发出的上下文请求有效</td></tr><tr><td>`fetch_cxt_ingress_head`</td><td>output</td><td>`TX_REQ_OOO_CXT_INGRESS_HEAD_WIDTH`</td><td>OoOStation</td><td>请求头部，含 QPN、操作码、通用头等</td></tr><tr><td>`fetch_cxt_ingress_data`</td><td>output</td><td>`TX_REQ_OOO_CXT_INGRESS_DATA_WIDTH`</td><td>OoOStation</td><td>请求数据，即锁存的 WQE 元数据</td></tr><tr><td>`fetch_cxt_ingress_start`</td><td>output</td><td>1</td><td>OoOStation</td><td>包起始标志</td></tr><tr><td>`fetch_cxt_ingress_last`</td><td>output</td><td>1</td><td>OoOStation</td><td>包结束标志</td></tr><tr><td>`fetch_cxt_ingress_ready`</td><td>input</td><td>1</td><td>OoOStation</td><td>OoOStation 准备好接收请求</td></tr></tbody></table>

## 状态机设计

该模块状态机较为简单，IDLE\_s和FETCH\_CXT\_s两个状态。

<table><thead><tr><th width="111">状态名</th><th width="87">编码</th><th width="608.714111328125">说明</th></tr></thead><tbody><tr><td>IDLE_s</td><td>2'd1</td><td>空闲状态：等待来自 WQEParser 的有效 WQE。此时 sub_wqe_ready = 1，允许接收新请求。</td></tr><tr><td>FETCH_CXT_s</td><td>2'd2</td><td>发送上下文请求状态：已锁存 WQE 元数据，正在向 OoOStation 发送上下文读取请求。此时 sub_wqe_ready = 0，拒绝新 WQE。</td></tr></tbody></table>

* 初始/复位状态：IDLE\_s
* IDLE\_s → FETCH\_CXT\_s\
  条件：sub\_wqe\_valid == 1\
  动作：锁存 sub\_wqe\_meta，提取 QPN，准备请求包。
* FETCH\_CXT\_s → IDLE\_s\
  条件：fetch\_cxt\_ingress\_valid && fetch\_cxt\_ingress\_ready == 1（即完成一次有效握手）\
  动作：释放流控，允许接收下一个 WQE。
