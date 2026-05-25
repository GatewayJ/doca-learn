# 13. 性能调优与开发排障手册

本章把各模块分散的性能与开发问题集中成一份实战清单。DOCA 性能调优不能脱离具体硬件、固件、驱动、DOCA 版本、网络拓扑和 workload；所有建议都应通过目标环境测量验证。

## 1. 调优前先建立事实基线

### 1.1 记录版本与拓扑

```bash
# DOCA 设备、库、能力
/opt/mellanox/doca/tools/doca_caps --list-devs
/opt/mellanox/doca/tools/doca_caps --list-rep-devs
/opt/mellanox/doca/tools/doca_caps --list-libs
/opt/mellanox/doca/tools/doca_caps

# PCI / NUMA / 网卡 / RDMA
lspci | grep -i mell
ibdev2netdev
rdma link
rdma dev
ip link show
ip addr show
```

建议把以下信息写进每次实验记录：

| 类别 | 记录项 |
|---|---|
| 硬件 | BlueField/SuperNIC/ConnectX 型号、端口速率、PCIe generation/width、NUMA node |
| 固件/驱动 | 固件版本、DOCA/DOCA-OFED 版本、内核版本、DPU OS 版本 |
| 网络 | MTU、VLAN、RoCE mode、GID index、PFC/ECN/DCQCN、交换机型号与配置 |
| 运行时 | CPU governor、IRQ affinity、hugepage、IOMMU、容器/裸机、权限 |
| workload | 包大小/IO size、并发、连接数、队列深度、读写比例、p50/p99/p999 |

### 1.2 不要混淆指标

| 指标 | 含义 | 容易误解 |
|---|---|---|
| Throughput | 吞吐，如 Gbps、IOPS、GB/s | 高吞吐可能牺牲尾延迟 |
| Latency | 单请求延迟 | 平均值没意义，至少看 p99 |
| CPU utilization | Host/DPU ARM CPU 消耗 | DPU ARM 打满也会导致系统抖动 |
| Drop/retry | 丢包、重传、RDMA retry | RoCE 配置问题常表现为尾延迟和 retry |
| Queue depth | outstanding 请求数 | 队列太深会放大排队延迟 |
| Rule scale | Flow 表项数 | 表项规模和 action 组合影响硬件资源 |

## 2. 通用性能方法论

1. **先跑官方 sample**：证明环境正确，再跑自写程序。
2. **一次只改一个变量**：MTU、queue depth、batch、CPU pinning 不要同时改。
3. **先测软件基线**：Host CPU 路径、DPDK/SPDK/RDMA perftest 基线都要有。
4. **区分 cold path 和 hot path**：建连、注册内存、下发规则是 cold path；转发/搬运/IO 是 hot path。
5. **复用昂贵资源**：mmap、MR、QP、Flow pipe、task pool 尽量复用。
6. **批量化但控制尾延迟**：batch 能提吞吐，但要看 p99/p999。
7. **先排环境再排代码**：RoCE/PCIe/固件/权限问题非常常见。

## 3. Host/DPU 系统级调优

| 项 | 建议 | 风险 |
|---|---|---|
| NUMA | 让应用线程、内存、NIC/DPU 所在 NUMA 节点尽量一致。 | NUMA 错位会导致额外 QPI/UPI 访问。 |
| CPU pinning | 高速 poll thread 固定 CPU，避免调度抖动。 | 固定不当会与 IRQ/SPDK reactor 抢核。 |
| CPU governor | 性能测试时使用 performance governor。 | 生产要平衡功耗和散热。 |
| Hugepage | DPDK/SPDK/大 buffer 场景优先使用 hugepage。 | 分配不足会导致程序启动失败。 |
| IOMMU | DMA 场景关注 IOMMU 模式和 passthrough 配置。 | 安全与性能取舍需按平台要求决策。 |
| IRQ affinity | 网络/RDMA 中断绑定到合适 CPU。 | 与 poll 模式混用要避免互相干扰。 |
| Thermal | DPU/NIC 温度过高会降频或异常。 | 长时间压测必须监控温度。 |

## 4. DOCA Core / 异步任务调优

| 方向 | 建议 |
|---|---|
| Task pool | 根据并发设置 max tasks，避免频繁 allocate/free。 |
| PE progress | 低延迟用专用 poll thread；省 CPU 用事件/reactive 模式。 |
| Callback | 回调只做轻量状态更新，把重活放到工作线程。 |
| Memory | 复用 `doca_mmap`/`doca_buf_inventory`，避免 hot path 注册/注销。 |
| Error path | 错误码、ctx state、resource id 必须进入日志。 |
| Threading | 不要假设所有 DOCA 对象都可任意跨线程并发访问；按官方线程安全说明和示例设计。 |

常见现象：

| 现象 | 可能原因 |
|---|---|
| task 提交后不完成 | 没有 progress PE、ctx 未 running、callback 未配置、设备不支持。 |
| start ctx 失败 | mandatory configuration 缺失、capability 不支持、设备/representor 选错。 |
| 性能不稳定 | callback 太重、poll thread 被抢占、资源频繁创建销毁。 |

## 5. Flow 调优

| 方向 | 建议 |
|---|---|
| Pipe 设计 | 先粗分类再细匹配，减少表项爆炸。 |
| Rule churn | 高频策略变化要批量处理，避免每包/每请求更新 Flow。 |
| Counter | 只给关键 pipe/entry 开 counter，避免耗尽监控资源。 |
| RSS/Queue | 根据 CPU/队列数设置 RSS，避免单队列热点。 |
| Miss path | 明确 miss 是 drop、next pipe、queue 还是 slow path。 |
| Aging | 大量短连接用 aging 回收表项，但要控制扫描/回调开销。 |
| Hairpin | 需要低 Host 往返时考虑 hairpin，但先验证硬件和 sample。 |

排障 checklist：

1. `doca_caps` 是否显示 flow installed/supported；
2. port/representor/uplink 是否选对；
3. pipe domain 是否符合流量方向；
4. match 字段是否真的出现在包里；
5. action 组合是否支持；
6. miss path 是否导致流量被 drop；
7. counter 是否命中；
8. Flow Inspector 导出的 pipeline 是否符合预期。

## 6. Comch 调优

Comch 是控制面，不是大数据通道。调优重点是稳定性、可观测性和背压。

| 方向 | 建议 |
|---|---|
| 消息格式 | 固定 header：magic/version/opcode/request_id/length/checksum。 |
| 批量 | 多个规则/资源变更合并成批量请求。 |
| 限流 | 每连接 outstanding request 上限，避免 DPU agent 被打爆。 |
| 超时 | 所有请求有 timeout 和可重试/不可重试分类。 |
| 版本 | HELLO 阶段协商协议版本和能力。 |
| 安全 | 校验租户、资源归属、payload 长度，拒绝未知 opcode。 |

常见问题：

- server 没启动或 service name 不一致；
- Host/DPU 使用不同协议版本；
- client 断开后 DPU 没释放资源；
- 控制面消息太大或太频繁；
- 日志没有 request_id，无法关联响应。

## 7. DMA 调优

| 方向 | 建议 |
|---|---|
| 数据大小 | 小块 copy 不一定值得走 DMA；大块/批量更适合。 |
| 注册复用 | hot path 不做 mmap register/unregister。 |
| 对齐 | buffer 地址和长度尽量按 cacheline/page/设备建议对齐。 |
| 并发 | 用多个 outstanding task 填满 DMA engine，但监控尾延迟。 |
| Pipeline | 与 RDMA/crypto/storage 形成流水线，减少 CPU 等待。 |
| Copy avoidance | 能 zero-copy 就不要多一次 DMA copy。 |

常见问题：

- src/dst `doca_buf` 属于不兼容设备或 mmap；
- 目标 buffer 太小或 data length 设置错；
- task 完成前释放 mmap/buf；
- CPU cache coherency/同步语义没处理好；
- DMA 引擎能力或最大 buffer 限制不满足。

## 8. RDMA/RoCE 调优

### 8.1 先用 perftest 建立网络基线

```bash
# Server
ib_write_bw

# Client
ib_write_bw <server>
```

同时检查：

```bash
ibdev2netdev
show_gids
rdma link
ip link show
```

### 8.2 关键参数

| 参数 | 影响 |
|---|---|
| MTU | 大包吞吐和包处理开销；两端和交换机要一致。 |
| GID index | 选择 RoCE 地址；错选会连接失败或走错网口。 |
| PFC/ECN/DCQCN | RoCE 拥塞和丢包控制；生产稳定性关键。 |
| QP 数 | 并发和资源占用；过多 QP 会增加管理和 cache 压力。 |
| CQ moderation | completion 合并；降低 CPU 但增加延迟。 |
| Inline size | 小消息 inline 可降低延迟，但受硬件限制。 |
| MR reuse | 内存注册昂贵，复用 MR/mmap。 |
| Queue depth | 决定 outstanding RDMA ops；过低吞吐不足，过高尾延迟高。 |

### 8.3 常见 RDMA 问题

| 现象 | 可能原因 |
|---|---|
| connect timeout | IP/GID/路由/防火墙/RDMA CM/交换机配置错误。 |
| 性能远低于线速 | MTU 小、PFC/ECN 错、NUMA 错、queue depth 低、CPU pinning 错。 |
| retry/timeout 增多 | RoCE 丢包、拥塞、PFC 配置不一致。 |
| remote access error | rkey/descriptor/权限/长度错误，或远端内存已释放。 |
| completion 语义误判 | 本端完成不等于应用层已消费，需要额外通知/同步。 |

## 9. SNAP / 存储路径调优

| 方向 | 建议 |
|---|---|
| Queue depth | 根据后端延迟和目标 IOPS 调整，不要只追求最大。 |
| 多队列 | 多核/多队列提升并发，但要绑定 CPU/队列亲和性。 |
| IO size | 4K 随机与大块顺序是完全不同 workload。 |
| Zero-copy | 减少 Host↔DPU↔backend 之间多余 copy。 |
| Backend | 先测 SPDK/NVMe-oF 后端基线，再接 SNAP。 |
| Flush/barrier | 数据一致性语义不能为了性能省略。 |
| Telemetry | 分层记录 Host queue、DPU queue、backend latency。 |

常见问题：

- Host 看到设备但 IO 卡住：controller/queue/reset/namespace 状态错误；
- 性能抖动：后端 RDMA 网络、SPDK reactor、DPU ARM 资源竞争；
- 数据一致性问题：flush、write ordering、cache policy 没设计；
- 多主机共享：缺 reservation/锁/集群 FS/后端一致性协议；
- reset/reconnect：Host reset 后 DPU 资源未正确重建。

## 10. Telemetry / 日志调优

| 方向 | 建议 |
|---|---|
| 分层指标 | Host、DPU agent、Flow、DMA/RDMA、backend 分开统计。 |
| 高基数控制 | tenant/resource/flow 标签不要无限增长。 |
| 采样 | 高频路径用采样或聚合，避免每包日志。 |
| Request ID | 控制面和数据面关键事件统一 request/resource id。 |
| 错误码 | 记录 DOCA error、errno、QP state、ctx state、PCI BDF。 |
| 安全 | 不记录密钥、rkey、token、完整内存地址。 |

## 11. 开发阶段常见问题总表

| 阶段 | 问题 | 处理方式 |
|---|---|---|
| 安装 | 找不到 `/opt/mellanox/doca` | 确认 DOCA package 安装位置和版本。 |
| 设备 | `doca_caps --list-devs` 为空 | 检查驱动、固件、权限、设备模式、是否在正确 Host/DPU 侧。 |
| 编译 | 找不到头文件/库 | 先 `pkg-config --list-all | grep -i doca`，再查安装路径，不硬猜 flags。 |
| 运行 | `unsupported` | 用 capability API 和 `doca_caps` 核对任务/设备/representor。 |
| 异步 | 没有 completion | 检查 PE progress、ctx state、callback、task 生命周期。 |
| 网络 | RDMA 连接失败 | 先用普通 RDMA 工具确认 RoCE，再看 DOCA RDMA。 |
| Flow | 规则安装成功但无流量 | 检查端口方向、representor、match 字段、miss、counter。 |
| 存储 | Host reset 后不可用 | 实现 reset/shutdown/recovery 状态机。 |
| 性能 | 压不上去 | 建基线，查 NUMA/MTU/queue depth/batch/CPU/DPU ARM。 |

## 12. 最小验证顺序

推荐每次开发按这个顺序推进：

1. `doca_caps` 能力探测；
2. 官方 reference app；
3. 最小 Hello/log 链接测试；
4. 单模块 sample：Flow/DMA/Comch/RDMA；
5. 两模块组合：Comch+DMA、Comch+Flow、Comch+RDMA；
6. 三段数据路径：Host↔DPU↔Remote；
7. 加入 telemetry；
8. 加入故障恢复；
9. 加入性能压测；
10. 最后才做生产化封装。

## 13. 延伸阅读

- [14. DOCA 代码实验语言选择与项目布局](14-code-lab-language-and-layout.md)
- [98. DOCA 技术术语表](98-technical-glossary.md)
- [99. NVIDIA DOCA 官方文档索引与扩展路线](99-official-reference-map.md)
