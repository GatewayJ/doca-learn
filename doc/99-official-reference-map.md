# 99. NVIDIA DOCA 官方文档索引与扩展路线

本页记录本系列当前引用过的官方文档入口，方便后续继续扩展。

## 1. 总入口与入门

| 主题 | 链接 | 用途 |
|---|---|---|
| DOCA Documentation v3.3.0 | <https://docs.nvidia.com/doca/sdk/index.html> | 总目录、版本、模块入口 |
| DOCA Developer Quick Start Guide | <https://docs.nvidia.com/doca/sdk/DOCA-Developer-Quick-Start-Guide/index.html> | 安装、访问 BlueField、运行参考应用 |
| Quick Start for BlueField Developers | <https://docs.nvidia.com/doca/sdk/Quick-Start-for-BlueField-Developers/index.html> | BlueField 开发者快速开始 |
| NVIDIA DOCA Developer Page | <https://developer.nvidia.com/doca> | 产品与下载入口 |

## 2. Core 与基础库

| 主题 | 链接 |
|---|---|
| DOCA Core | <https://docs.nvidia.com/doca/sdk/DOCA-Core/index.html> |
| DOCA Common | <https://docs.nvidia.com/doca/sdk/DOCA-Common/index.html> |
| DOCA Log | <https://docs.nvidia.com/doca/sdk/DOCA-Log/index.html> |
| DOCA Arg Parser | 在总目录中查找 Arg Parser 相关页面 |

重点章节：

- Architecture；
- DOCA Device；
- DOCA Memory Subsystem；
- DOCA Execution Model；
- DOCA Context；
- DOCA Task；
- DOCA Progress Engine；
- Object Life Cycle。

## 3. 网络数据面

| 主题 | 链接 |
|---|---|
| DOCA Flow | <https://docs.nvidia.com/doca/sdk/DOCA-Flow/index.html> |
| DOCA Flow Connection Tracking | <https://docs.nvidia.com/doca/sdk/DOCA-Flow-Connection-Tracking/index.html> |
| DOCA Flow Tune Server | <https://docs.nvidia.com/doca/sdk/DOCA-Flow-Tune-Server/index.html> |
| DOCA Flow Inspector Service Guide | <https://docs.nvidia.com/doca/sdk/DOCA-Flow-Inspector-Service-Guide/index.html> |
| Flow Control | <https://docs.nvidia.com/doca/sdk/Flow-Control/index.html> |
| Flow Steering | <https://docs.nvidia.com/doca/sdk/Flow-Steering/index.html> |

重点章节：

- Steering Domains；
- Flow Life Cycle；
- Pipe Mode；
- Pipe Entry；
- Forwarding / Miss；
- Monitoring / Counters；
- Hardware Steering Mode；
- Teardown。

## 4. Host-DPU 通信

| 主题 | 链接 |
|---|---|
| DOCA Comch | <https://docs.nvidia.com/doca/sdk/DOCA-Comch/index.html> |
| DOCA DPA Comms | <https://docs.nvidia.com/doca/sdk/DOCA-DPA-Comms/index.html> |
| DOCA Comm Channel Admin Tool | <https://docs.nvidia.com/doca/sdk/DOCA-Comm-Channel-Admin-Tool/index.html> |

重点章节：

- Server/client initialization flow；
- Control Channel Send Task；
- Connection Status Changed Event；
- Security Considerations；
- DPA support。

## 5. DMA / RDMA / RoCE

| 主题 | 链接 |
|---|---|
| DOCA DMA | <https://docs.nvidia.com/doca/sdk/DOCA-DMA/index.html> |
| DOCA DMA Copy Application Guide | <https://docs.nvidia.com/doca/sdk/DOCA-DMA-Copy-Application-Guide/index.html> |
| DOCA RDMA | <https://docs.nvidia.com/doca/sdk/DOCA-RDMA/index.html> |
| DOCA RDMA Verbs | <https://docs.nvidia.com/doca/sdk/DOCA-RDMA-Verbs/index.html> |
| RDMA over Converged Ethernet | <https://docs.nvidia.com/doca/sdk/RDMA-over-Converged-Ethernet/index.html> |
| DOCA Storage Target RDMA Application Guide | <https://docs.nvidia.com/doca/sdk/DOCA-Storage-Target-RDMA-Application-Guide/index.html> |
| DOCA Storage Comch to RDMA Zero Copy Application Guide | <https://docs.nvidia.com/doca/sdk/DOCA-Storage-ComCh-to-RDMA-Zero-copy-Application-Guide/index.html> |
| DOCA Storage Comch to RDMA GGA Offload Application Guide | <https://docs.nvidia.com/doca/sdk/DOCA-Storage-Comch-to-RDMA-GGA-Offload-Application-Guide/index.html> |

重点章节：

- Objects；
- Device；
- Memory Map；
- Buffer Inventory and Buffers；
- Configuration Phase；
- Execution Phase；
- Tasks；
- Events；
- State Machine；
- Alternative Datapath Options。

## 6. 存储、SNAP、设备仿真

| 主题 | 链接 |
|---|---|
| Storage Protocols | <https://docs.nvidia.com/doca/sdk/Storage-Protocols/index.html> |
| DOCA Storage Applications | <https://docs.nvidia.com/doca/sdk/DOCA-Storage-Applications/index.html> |
| DOCA SNAP Services | <https://docs.nvidia.com/doca/sdk/DOCA-SNAP-Services/index.html> |
| DOCA SNAP-4 Service Guide | <https://docs.nvidia.com/doca/sdk/DOCA-SNAP-4-Service-Guide/index.html> |
| DOCA SNAP-3 User Guide | <https://docs.nvidia.com/doca/sdk/DOCA-SNAP-3-User-Guide/index.html> |
| DOCA NVMe Emulation Application Guide | <https://docs.nvidia.com/doca/sdk/DOCA-NVMe-Emulation-Application-Guide/index.html> |
| DOCA DevEmu Virtio | <https://docs.nvidia.com/doca/sdk/DOCA-DevEmu-Virtio/index.html> |
| DOCA DevEmu Virtio-fs | <https://docs.nvidia.com/doca/sdk/DOCA-DevEmu-Virtio-FS/index.html> |
| NVMe-oF | <https://docs.nvidia.com/doca/sdk/NVME-oF---NVM-Express-over-Fabrics/index.html> |

## 7. 安全、数据处理与遥测

| 主题 | 链接 |
|---|---|
| DOCA App Shield | <https://docs.nvidia.com/doca/sdk/DOCA-App-Shield/index.html> |
| DOCA App Shield Agent Application Guide | <https://docs.nvidia.com/doca/sdk/DOCA-App-Shield-Agent-Application-Guide/index.html> |
| DOCA AES-GCM | <https://docs.nvidia.com/doca/sdk/DOCA-AES-GCM/index.html> |
| DOCA SHA | <https://docs.nvidia.com/doca/sdk/DOCA-SHA/index.html> |
| DOCA Compress | <https://docs.nvidia.com/doca/sdk/DOCA-Compress/index.html> |
| DOCA Erasure Coding | 在总目录中查找 DOCA Erasure Coding |
| DOCA Telemetry | <https://docs.nvidia.com/doca/sdk/DOCA-Telemetry/index.html> |
| DOCA Telemetry Exporter | <https://docs.nvidia.com/doca/sdk/DOCA-Telemetry-Exporter/index.html> |
| DOCA Telemetry Service Guide | <https://docs.nvidia.com/doca/sdk/DOCA-Telemetry-Service-Guide/index.html> |

## 8. DPA / GPU / AI 网络相关

| 主题 | 链接 |
|---|---|
| DPA Subsystem | <https://docs.nvidia.com/doca/sdk/DPA-Subsystem/index.html> |
| DPA Development | <https://docs.nvidia.com/doca/sdk/DPA-Development/index.html> |
| DOCA DPA | <https://docs.nvidia.com/doca/sdk/DOCA-DPA/index.html> |
| DOCA DPA Verbs | <https://docs.nvidia.com/doca/sdk/DOCA-DPA-Verbs/index.html> |
| DOCA DPACC Compiler | <https://docs.nvidia.com/doca/sdk/DOCA-DPACC-Compiler/index.html> |
| DOCA GPUNetIO | <https://docs.nvidia.com/doca/sdk/DOCA-GPUNetIO/index.html> |
| GPUNetIO Architecture and Design | <https://docs.nvidia.com/doca/sdk/GPUNetIO-Architecture-and-Design/index.html> |
| DOCA PCC | <https://docs.nvidia.com/doca/sdk/DOCA-PCC/index.html> |

## 9. 本系列补充文档与后续扩展

本系列已经新增三份横向参考文档：

- [13-performance-tuning-and-troubleshooting.md](13-performance-tuning-and-troubleshooting.md)：性能调优、压测方法和开发排障手册；
- [14-code-lab-language-and-layout.md](14-code-lab-language-and-layout.md)：代码实验语言选择、labs 目录布局、C/Python/C++/Go/Rust 分工；
- [98-technical-glossary.md](98-technical-glossary.md)：DOCA/Flow/Comch/DMA/RDMA/SNAP/RoCE 等术语集中解释。

本系列已经新增代码实验章节：

- [15-lab-00-hello-log-c.md](15-lab-00-hello-log-c.md)：Hello DOCA Log，验证头文件、链接和日志；
- [16-lab-01-caps-python.md](16-lab-01-caps-python.md)：Python 环境探测与 `doca_caps` 输出归档；
- [17-lab-02-dma-local-copy-c.md](17-lab-02-dma-local-copy-c.md)：DMA local copy 实验步骤；
- [18-lab-03-comch-host-dpu-c.md](18-lab-03-comch-host-dpu-c.md)：Host/DPU Comch 控制协议 demo；
- [19-lab-04-rdma-write-read-c.md](19-lab-04-rdma-write-read-c.md)：RDMA write/read 实验步骤；
- [20-lab-05-flow-acl-c.md](20-lab-05-flow-acl-c.md)：Flow ACL + counter 实验。

后续可以继续新增更偏进阶/生产化的文档：

1. `21-roce-debugging-runbook.md`：RoCE 网络排障手册；
2. `22-production-checklist.md`：生产化检查清单；
3. `23-cpp-raii-wrapper.md`：C++ RAII 封装设计；
4. `24-control-plane-service.md`：Go/Rust/Python 外围控制面设计。

## 10. 官方文档阅读方法

阅读 NVIDIA DOCA 文档时建议按这个顺序：

1. Introduction：确认模块解决什么问题；
2. Prerequisites / Environment：确认硬件和软件前提；
3. Architecture / Objects：建立对象关系；
4. Configuration Phase：看 mandatory configuration；
5. Execution Phase / Tasks：看异步任务和事件；
6. State Machine：看 start/stop/error；
7. Samples / Application Guide：跑官方示例；
8. API Reference：最后再查具体函数签名。
