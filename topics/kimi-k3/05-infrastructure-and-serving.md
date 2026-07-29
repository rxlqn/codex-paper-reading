# 专题 5：3T 训练、1M RL 与在线推理基础设施

## 核心判断

K3 的规模改变了三个系统约束：

- 2.8T MoE：expert imbalance 会让大量设备等待；
- 1M agentic RL：最贵的不只是生成 token，而是丢失长轨迹的模型缓存和 sandbox 状态；
- Hybrid KDA-MLA serving：固定大小 recurrent state 与随序列增长的 KV cache 必须联合管理。

报告 §5 的价值在于说明这些约束如何反过来影响算法和架构。

## 1. KDA 的系统协同

KDA 在序列方向是 recurrent 的，但 chunk 内可并行。系统需要覆盖：

- 短序列/长序列的不同 kernel regime；
- context parallel ranks 之间的 recurrent state 传播；
- lower-bounded decay 带来的 dense Tensor Core 路径；
- decode 时 recurrent state 的原地更新。

KDA Context Parallelism 不能像普通 full attention 一样只切 KV。每个 rank 既要计算本地 token 贡献，又要接收之前 ranks 累积的 state。报告利用 recurrence 的组合结构，把与 incoming state 无关的局部部分先算，从而隐藏跨 rank 依赖。

## 2. 3T MoE Pre-Training

训练并行组合包括：

- Pipeline Parallelism 与 virtual stages；
- Expert Parallelism；
- Tensor Parallelism；
- Sequence Parallelism；
- ZeRO-style Data Parallelism。

### MoonEP

极稀疏 MoE 的常见问题是某些 experts 过热，导致 EP ranks 收到的 token 数不同。动态 shape 会进一步破坏 kernel 和 communication efficiency。

MoonEP 的目标：

- 每个 rank 接收完全相同数量的 tokens；
- 用 bounded redundant experts 处理局部负载；
- planning kernel 预计算 token destination；
- permute/unpermute 使用 zero-copy communication；
- 所有层拥有静态计算 shape。

Quantile Balancing 从路由侧减少 imbalance，MoonEP 从执行侧保证 balance，两者共同工作。

### 显存管理

2.8T native multimodal training 还需处理：

- activation offloading；
- communication/computation/offloading overlap；
- memory-efficient optimizer states；
- vision encoder 的 activation 与 gradient；
- pipeline bubble 和 multimodal token 长度差异。

报告描述了机制，但缺少总 GPU 数、MFU、训练时长、故障率和总能耗，因此尚不能完成成本复算。

## 3. 1M Agentic RL

K3 使用 co-located RL，让 rollout generation 与 training 共享资源，把单个 1M-context experiment 控制在数百 GPU。

### External KV cache pool

长轨迹跨 iteration 暂停后，如果 KV 丢失，恢复时要重新 prefill 巨大 prefix。external pool 把 KV 从短生命周期 worker 中移出，使 partial rollout 可恢复。

### Adaptive throttling

generation 和 training 的资源需求随 rollout 到达动态变化。固定配额会让一侧空闲、一侧拥塞，因此系统根据队列和负载调整两者节奏。

### Resumable sandbox

模型 KV 只代表对话状态，agent 的真实环境还在 microVM。恢复需要 model-side prefix 与 environment-side snapshot 对齐，否则 transcript 说“文件已生成”，实际文件系统却不存在。

## 4. Hybrid Cache

KDA 与 MLA 有两种不同 cache：

- MLA KV cache：随 token 数增长，按 token 分页；
- KDA state：大小固定，但每个请求只有一份当前 recurrent state。

K3 把两者装进统一 paged block pool，共享 allocation、reference counting 和 eviction。

### 解耦物理块与 hash 粒度

KDA checkpoint 很大，不能每个 token 保存；物理 block 因此可能达到 1024-6144 tokens。若 prefix hash 也绑定同一粒度，短请求几乎无法命中。

解决方案：

- physical block 保持粗粒度；
- prefix hash 在块内按更细的 512-token endpoint；
- KDA checkpoint 只保存在部分 hash endpoints；
- lookup 要同时命中 MLA prefix 和所有 KDA cache groups 的 checkpoint；
- hit 后 copy-on-write，避免共享 checkpoint 被请求原地修改。

这使 2800-token 匹配可以在 2560-token 边界恢复，而不必回退到 0 或 6144。

## 5. Speculative Decoding

KDA decode 原地更新 state。若 speculative draft 部分被拒绝，state 已越过最后 accepted token，直接 rollback 很贵。

K3 不保存每个 draft position 的完整 state，而只缓存更小的 projected inputs；确认 accepted prefix 后，在 on-chip recurrent loop 中重建 state。这个设计让 verification latency 随 draft token 数次线性增长，并避免巨大的 state traffic。

Draft model 来自预训练 MTP layer，并微调为 EAGLE-3-style 单层 draft。输入融合第 1、第 4 和最后 AttnRes block 的特征，训练直接优化 speculative acceptance rate 的 likelihood-based loss，而不只做 KL surrogate。

## 6. Fleet Scheduling

### Cache-aware affinity

典型 coding 请求可能带 400K prefix，只新增 4K。cache miss 需要重做 400K prefill，因此 session 优先路由到持有 prefix 的 cluster。

为降低单 cluster 故障风险：

- consistent hashing 为 session 指定 primary 和 secondary；
- 平时 primary 服务；
- secondary 不复制完整 cache，故障时重新 prefill；
- secondary assignments 在 fleet 内分散，避免 failover storm 集中。

### Budget-based admission

请求成本从 <2K 到 1M，跨约三个数量级。按“平均请求”限流会让一批超长请求拖垮所有短请求。K3 为不同 request class 分配独立 resource budget，隔离 burst。

## 部署含义

“开放权重”不等于“普通团队可本地部署”：

- 2.8T total parameters 即使 MXFP4 也需要极大权重存储；
- 104B active parameters 决定每 token 计算仍然很高；
- 1M context 的 cache、network 和 scheduling 是集群级问题；
- 官方推荐 vLLM、SGLang、TokenSpeed，但可启动不代表能达到报告中的吞吐和成本。

更现实的开放价值可能是：

- API 可用性；
- 大型推理供应商部署；
- 研究小型 architecture proxy；
- 蒸馏、量化和系统组件复用。

## 未回答问题

- 预训练集群规模、训练天数、MFU、故障恢复和总能耗是多少？
- MoonEP 的端到端收益相对成熟 EP stack 有多大？
- external KV pool 的容量、带宽、命中率和恢复延迟怎样？
- 1M 请求的 TTFT、TPOT、并发和单位成本是多少？
- secondary cluster failover 的 re-prefill 会怎样影响尾延迟？
- MXFP4/MXFP8 在不同硬件上的实际兼容性和精度差异如何？
