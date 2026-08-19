# docatest

`docatest` 是一个中文 DOCA 学习文档项目，目标是参考 NVIDIA 官方 DOCA 文档，把 DOCA 从应用场景、平台组成、底层原理、开发模型到 RDMA/DMA/Flow/Comch/SNAP 等关键技术串成一套从零到一的学习路线。

当前文档基于已核对的 NVIDIA 官方文档入口：

- DOCA Documentation v3.3.0: <https://docs.nvidia.com/doca/sdk/index.html>
- DOCA Developer Quick Start Guide: <https://docs.nvidia.com/doca/sdk/DOCA-Developer-Quick-Start-Guide/index.html>
- DOCA Core: <https://docs.nvidia.com/doca/sdk/DOCA-Core/index.html>
- DOCA Flow: <https://docs.nvidia.com/doca/sdk/DOCA-Flow/index.html>
- DOCA DMA: <https://docs.nvidia.com/doca/sdk/DOCA-DMA/index.html>
- DOCA RDMA: <https://docs.nvidia.com/doca/sdk/DOCA-RDMA/index.html>
- DOCA Comch: <https://docs.nvidia.com/doca/sdk/DOCA-Comch/index.html>
- DOCA Capabilities Print Tool: <https://docs.nvidia.com/doca/sdk/DOCA-Capabilities-Print-Tool/index.html>

## 阅读入口

从 `doc/00-series-index.md` 开始：

- [00-series-index.md](doc/00-series-index.md) — 系列总览与学习路线
- [01-what-is-doca-and-use-cases.md](doc/01-what-is-doca-and-use-cases.md) — DOCA 是什么、解决什么问题、典型应用场景
- [02-platform-components-and-environment.md](doc/02-platform-components-and-environment.md) — BlueField/SuperNIC、Host、DPU OS、SDK、服务与工具组成
- [03-architecture-and-offload-principles.md](doc/03-architecture-and-offload-principles.md) — Host/DPU ARM/硬件快路径的分工原理
- [04-doca-core-programming-model.md](doc/04-doca-core-programming-model.md) — DOCA Core 编程模型：device、mmap、buf、ctx、task、PE
- [05-doca-flow-network-datapath.md](doc/05-doca-flow-network-datapath.md) — DOCA Flow 网络数据面与硬件转发管线
- [06-doca-comch-control-channel.md](doc/06-doca-comch-control-channel.md) — DOCA Comch：Host↔DPU 控制通道
- [07-doca-dma-memory-movement.md](doc/07-doca-dma-memory-movement.md) — DOCA DMA：本地/跨端内存搬运
- [08-doca-rdma-and-roce.md](doc/08-doca-rdma-and-roce.md) — DOCA RDMA 与 RoCE：远端内存访问
- [09-storage-snap-nvme-virtio.md](doc/09-storage-snap-nvme-virtio.md) — DOCA 存储、SNAP、NVMe/virtio 设备仿真
- [10-security-crypto-telemetry-ops.md](doc/10-security-crypto-telemetry-ops.md) — 安全、加密、压缩、遥测与运维工具
- [11-hands-on-quickstart-and-first-app.md](doc/11-hands-on-quickstart-and-first-app.md) — 从零实操：环境检查、能力探测、第一个参考应用
- [12-end-to-end-rdma-flow-storage-design.md](doc/12-end-to-end-rdma-flow-storage-design.md) — 综合设计：Comch + Flow + DMA/RDMA + Storage 原型
- [13-performance-tuning-and-troubleshooting.md](doc/13-performance-tuning-and-troubleshooting.md) — 性能调优与开发排障手册
- [14-code-lab-language-and-layout.md](doc/14-code-lab-language-and-layout.md) — 代码实验语言选择与项目布局
- [15-lab-00-hello-log-c.md](doc/15-lab-00-hello-log-c.md) — Lab 00：Hello DOCA Log（C）
- [16-lab-01-caps-python.md](doc/16-lab-01-caps-python.md) — Lab 01：Capabilities Python 环境探测
- [17-lab-02-dma-local-copy-c.md](doc/17-lab-02-dma-local-copy-c.md) — Lab 02：DMA Local Copy（C）
- [18-lab-03-comch-host-dpu-c.md](doc/18-lab-03-comch-host-dpu-c.md) — Lab 03：Comch Host/DPU 控制通道（C）
- [19-lab-04-rdma-write-read-c.md](doc/19-lab-04-rdma-write-read-c.md) — Lab 04：RDMA Write/Read（C）
- [20-lab-05-flow-acl-c.md](doc/20-lab-05-flow-acl-c.md) — Lab 05：Flow ACL + Counter（C）
- [21-doca-dpa-flexio-programming-model.md](doc/21-doca-dpa-flexio-programming-model.md) — DPA、FlexIO 与 DPACC 编程模型
- [22-doca-gpunetio-gpu-packet-processing.md](doc/22-doca-gpunetio-gpu-packet-processing.md) — GPUNetIO 与 GPU 直接处理网络包
- [98-technical-glossary.md](doc/98-technical-glossary.md) — DOCA 技术术语表
- [99-official-reference-map.md](doc/99-official-reference-map.md) — 官方文档索引与后续扩展路线

## 说明

这些文档不是 NVIDIA 官方文档的替代品，而是中文学习路径和架构笔记。涉及安装包名、PCI 地址、设备能力、命令路径时，以目标机器上安装的 DOCA 版本和官方文档为准。
