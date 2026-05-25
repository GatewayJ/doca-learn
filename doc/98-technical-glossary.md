# 98. DOCA 技术术语表

本页把系列中反复出现的技术名词集中解释。它不是 API Reference 的替代品；具体函数签名、支持矩阵、参数范围请以目标 DOCA 版本官方文档和本机 `doca_caps` 输出为准。

## 1. 平台与硬件

| 术语 | 解释 | 开发时要注意 |
|---|---|---|
| DPU | Data Processing Unit。NVIDIA BlueField 这类设备通常包含 ARM CPU、NIC 数据面、PCIe 互联和硬件加速能力。 | DPU ARM 不是无限资源；控制面和慢路径仍会消耗 CPU。 |
| BlueField | NVIDIA DPU 产品线。可运行 DPU OS，并通过 DOCA 编程网络、存储、安全等基础设施功能。 | 型号、固件、工作模式会影响支持的 DOCA 能力。 |
| SuperNIC | 面向 AI/云网络的 NVIDIA 高性能智能网卡产品线，部分能力可通过 DOCA 使用。 | 不要假设所有 BlueField 能力在 SuperNIC 上都存在。 |
| Host | 插有 DPU/NIC 的服务器主机，运行 Linux、hypervisor、VM/container、业务应用。 | Host 与 DPU 侧命令、PCI 地址、设备名经常不同。 |
| DPU OS | BlueField ARM 侧运行的 Linux/运行时环境。 | Host 上装了 DOCA 不等于 DPU 侧也装好；两侧版本要对齐。 |
| PCI BDF | PCI Bus:Device.Function，例如 `0000:08:00.0`。 | 官方示例 PCI 地址不能照抄，必须用 `doca_caps --list-devs` 查询。 |
| PF/VF/SF | Physical Function / Virtual Function / Subfunction。用于 SR-IOV、虚拟化、隔离和资源切分。 | Flow/representor/RDMA 能力常与 PF/VF/SF 角色有关。 |
| Representor | DPU/OVS/eSwitch 中代表某个 Host VF/SF/端口的控制面网络接口。 | 做 Flow 转发时，错选 representor 是常见问题。 |
| Uplink | 物理上联端口或其 representor。 | 区分 uplink 与 Host-side representor，避免流量方向反了。 |
| eSwitch | NIC/DPU 内部以太交换与 steering 能力。 | Flow、representor、SR-IOV 场景常依赖 eSwitch 配置。 |
| BAR / PCIe window | Host 与设备之间暴露寄存器或内存窗口的 PCIe 机制。 | DMA/设备仿真场景会受 PCIe 拓扑、IOMMU、权限影响。 |

## 2. DOCA Core 对象

| 术语 | 解释 | 常见坑 |
|---|---|---|
| `doca_devinfo` | 设备信息对象，用于枚举和查询能力。 | 只能说明“可能可用”，真正使用前仍要打开 `doca_dev` 并检查能力。 |
| `doca_dev` | 打开后的 DOCA device handle。 | Host/DPU 侧可见设备不同；不要混用 BDF。 |
| Capability query | 通过模块 capability API 或 `doca_caps` 查询是否支持某 task/限制。 | 不查询能力就写代码，最容易遇到 `unsupported` 或 start 失败。 |
| `doca_mmap` | DOCA 内存映射/注册对象，使设备能够访问某段内存。 | 生命周期必须覆盖所有 task；权限和设备列表要正确。 |
| `doca_buf_inventory` | `doca_buf` 描述符池。 | 容量不足会导致无法创建 buffer；高并发需预估。 |
| `doca_buf` | 描述一段已注册内存的 buffer，包括地址、长度、数据范围等。 | buffer 长度、data length、offset 错误会导致 task 失败或数据不完整。 |
| `doca_ctx` | 模块统一上下文，承载 start/stop/state/task/event。 | 必须完成 mandatory configuration 后再 start。 |
| `doca_task` | 异步任务抽象，例如 DMA copy、RDMA write、Comch send。 | 提交后要通过 PE 推进；task 完成前不要释放相关资源。 |
| `doca_pe` | Progress Engine，推进异步 task/event 并触发 callback。 | 忘记 `doca_pe_progress()` 会表现为“任务卡住”。 |
| Callback | task completion/error 回调。 | 回调中不要做重活；避免死锁、长时间阻塞和递归提交混乱。 |
| `doca_error_t` | DOCA 错误码类型。 | 记录错误码和 `doca_error_get_descr()` 类信息，便于排障。 |

## 3. DOCA Flow 术语

| 术语 | 解释 | 调优/排障提示 |
|---|---|---|
| Port | Flow 端口抽象，连接实际 netdev/representor/uplink 等。 | 端口方向错会导致规则安装成功但无流量命中。 |
| Pipe | 一段 match-action 处理模板或阶段。 | 将规则组织成多级 pipe，可减少表项膨胀。 |
| Pipe entry | 具体规则实例。 | 批量插入/删除比高频单条抖动更稳定。 |
| Match | 匹配字段，例如 L2/L3/L4、tunnel、metadata。 | 支持字段以硬件能力为准。 |
| Action | drop、forward、modify、encap/decap、count 等动作。 | 某些 action 组合可能不被目标硬件支持。 |
| Fwd / Miss | 命中转发目标和未命中路径。 | miss 路径不清会造成黑洞或意外 slow path。 |
| Counter / Monitor | 计数、aging、meter 等观测/控制能力。 | 计数器有资源成本，关键路径选择性开启。 |
| RSS | Receive Side Scaling，把流量分散到多个队列。 | 调优队列数、CPU affinity、hash 字段。 |
| Hairpin | NIC 内部队列/端口间转发，减少 Host 往返。 | 配置复杂，先跑官方 sample/最小规则。 |
| Flow CT | Connection Tracking，连接跟踪能力。 | 需要理解状态表规模、超时、老化和异常路径。 |

## 4. Comch / 控制通道术语

| 术语 | 解释 | 常见坑 |
|---|---|---|
| Comch server | 通常运行在 DPU 侧，监听 service name。 | service name、设备、权限错会连接失败。 |
| Comch client | 通常运行在 Host 侧，连接 server。 | Host 与 DPU 两侧版本协议要协商。 |
| Control message | 小型命令/状态/元数据消息。 | 不适合搬大块数据，大数据走 DMA/RDMA/设备队列。 |
| Request ID | 请求唯一 ID，用于关联响应和日志。 | 没有 request_id 会让并发排障困难。 |
| Producer/Consumer / MsgQ | Comch 中面向更复杂通信模式/DPA 通信的对象。 | 先掌握 server/client send task，再扩展。 |
| Backpressure | 消费速度低于生产速度时的限流。 | 控制通道也需要队列上限和超时。 |

## 5. DMA / RDMA / RoCE 术语

| 术语 | 解释 | 常见坑 |
|---|---|---|
| DMA | Direct Memory Access，本地/设备可访问内存间搬运。 | 内存必须注册并可被设备访问。 |
| RDMA | Remote DMA，跨网络访问远端已注册内存。 | 远端内存需要注册、授权、交换 descriptor/key。 |
| RoCE | RDMA over Converged Ethernet。 | PFC/ECN/MTU/GID/交换机配置是稳定性的关键。 |
| QP | Queue Pair，RDMA send/recv/read/write 的队列实体。 | QP 数量、状态、路径 MTU、retry 会影响性能和稳定性。 |
| CQ | Completion Queue，RDMA completion 队列。 | CQ 轮询/中断、CQ moderation 会影响延迟和 CPU。 |
| MR | Memory Region，已注册内存区域。 | 注册/注销昂贵，应复用。 |
| lkey/rkey | 本地/远端内存访问 key。 | 泄露 rkey 等于暴露远端内存能力，必须保护。 |
| GID | RoCE 全局 ID，用于地址选择。 | GID index 错是 RoCE 连接失败高频原因。 |
| RC/UC/UD | Reliable Connected / Unreliable Connected / Unreliable Datagram。 | 语义不同，存储/可靠传输通常关注 RC。 |
| PFC | Priority Flow Control，以太网无损优先级流控。 | 配错会导致丢包或拥塞扩散。 |
| ECN/DCQCN | RoCE 拥塞标记与拥塞控制机制。 | 生产 RoCE 通常需要交换机和端侧统一配置。 |

## 6. 存储与设备仿真术语

| 术语 | 解释 | 常见坑 |
|---|---|---|
| SNAP | DPU 侧存储虚拟化/设备仿真服务。 | Host 看到标准设备，不代表后端一致性自动解决。 |
| DevEmu | Device Emulation，设备仿真相关能力。 | reset/hotplug/error recovery 必须设计。 |
| NVMe Controller | NVMe 控制器抽象，Host 通过寄存器/队列与其交互。 | controller reset、shutdown、admin queue 要正确处理。 |
| Namespace | NVMe 命名空间，类似一块逻辑盘。 | 多 controller/多 writer 共享需要一致性协议或 reservation。 |
| SQ/CQ | Submission Queue / Completion Queue。 | queue depth、队列数、CPU affinity 直接影响性能。 |
| PRP/SGL | NVMe 描述数据 buffer 的机制。 | DPU 侧要正确解析并访问 Host buffer。 |
| SPDK bdev | SPDK 块设备抽象。 | 多消费者一致性取决于 bdev 类型和上层协议。 |
| NVMe-oF | NVMe over Fabrics，常跑在 RDMA/TCP 上。 | RDMA transport 要先用 perftest/官方工具确认稳定。 |
| virtio | 虚拟化标准设备接口，如 virtio-blk、virtio-fs、virtio-net。 | virtqueue、feature negotiation、reset 处理要完整。 |

## 7. 性能与运维术语

| 术语 | 解释 | 使用建议 |
|---|---|---|
| Queue depth | 并发 outstanding 请求数量。 | 太低吞吐上不去，太高增加尾延迟和内存占用。 |
| Batch | 批量提交/完成处理。 | 提升吞吐，但可能增加单请求延迟。 |
| Polling | 忙轮询 completion。 | 低延迟但耗 CPU；生产需隔离核心。 |
| Interrupt/Event | 事件/中断驱动。 | 省 CPU，但延迟通常更高。 |
| NUMA locality | CPU、内存、PCIe 设备的拓扑亲和性。 | Host 上 DPU/NIC 所在 NUMA 节点应与应用/内存尽量一致。 |
| Hugepage | 大页内存，减少 TLB 压力，常用于 DPDK/SPDK。 | 分配、权限、mount 要提前准备。 |
| Telemetry cardinality | 指标标签组合数量。 | 高基数指标会压垮监控系统。 |
| SLO | 服务目标，例如 p99 latency、IOPS、drop rate。 | 调优前先定义目标和基线。 |

## 8. 延伸阅读

- [13. 性能调优与开发排障手册](13-performance-tuning-and-troubleshooting.md)
- [99. NVIDIA DOCA 官方文档索引与扩展路线](99-official-reference-map.md)
