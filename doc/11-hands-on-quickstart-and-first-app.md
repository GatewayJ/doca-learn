# 11. 从零实操：环境检查、能力探测与第一个参考应用

> 本章给出真实 DOCA 环境中的操作路径。命令不会假设当前机器已经有 DPU；在你的 BlueField/SuperNIC 机器上执行前，请先核对官方文档和本机安装路径。

## 1. 官方 Quick Start 的核心路径

NVIDIA DOCA Developer Quick Start Guide 的目标是：

1. 安装 BlueField Networking Platform；
2. 安装 DOCA Software Package；
3. 访问 BlueField；
4. 运行 DOCA reference application。

官方文档示例中，Host 访问 BlueField 的命令形态为：

```bash
host# ssh ubuntu@192.168.100.2
```

实际 IP、用户名、认证方式以你的 BlueField 配置为准。

## 2. 能力探测

先不要写代码，先确认设备和库。

### 2.1 列出设备

```bash
/opt/mellanox/doca/tools/doca_caps --list-devs
```

关注字段：

- PCI 地址，例如 `0000:08:00.0`；
- `ibdev_name`，例如 `mlx5_0`；
- `iface_name`，例如 `eth2`；
- `pci_func_type`，PF/VF/SF；
- MAC/IP；
- uplink port。

### 2.2 列出 representor

```bash
/opt/mellanox/doca/tools/doca_caps --list-rep-devs
```

在虚拟化/Flow 场景中，representor 关系非常关键。

### 2.3 列出库

```bash
/opt/mellanox/doca/tools/doca_caps --list-libs
```

确认你要学的库是否 installed，例如：

- common/core/log/argp；
- flow；
- dma；
- rdma；
- comch；
- aes_gcm / sha / compress；
- telemetry；
- snap/devemu 相关服务按安装包而定。

### 2.4 打印详细能力

```bash
/opt/mellanox/doca/tools/doca_caps
```

关注：

- task supported/unsupported；
- max task 数；
- max buffer size；
- mmap export/import 能力；
- Flow action/match/domain 支持；
- RDMA send/read/write/atomic 支持。

## 3. 运行官方 Secure Channel 参考应用

官方 Developer Quick Start Guide 使用 Secure Channel 作为参考应用。

DPU/server 侧示例：

```bash
/opt/mellanox/doca/applications/secure_channel/bin/doca_secure_channel \
  -s 256 \
  -n 10 \
  -p 03:00.0 \
  -r 3b:00.0
```

Host/client 侧示例：

```bash
/opt/mellanox/doca/applications/secure_channel/bin/doca_secure_channel \
  -s 256 \
  -n 10 \
  -p 3b:00.0
```

注意：

- `03:00.0`、`3b:00.0` 是示例 PCI 地址；
- 你的机器必须用 `doca_caps --list-devs` 和 `--list-rep-devs` 确认实际地址；
- 先理解每个参数含义再改；
- 如果失败，先看设备、库、权限、固件/驱动版本，不要直接改代码。

## 4. 最小 Hello DOCA：只验证 SDK 链接

如果你想先验证头文件和库链接，可写一个最小 DOCA Log 程序。

`hello_doca.c`：

```c
#include <stdio.h>
#include <doca_log.h>

DOCA_LOG_REGISTER(HELLO_DOCA);

int main(int argc, char **argv)
{
    doca_log_backend_create_standard();
    DOCA_LOG_INFO("Hello, DOCA!");
    printf("Hello from normal printf too.\n");
    return 0;
}
```

先探索 pkg-config：

```bash
pkg-config --list-all | grep -i doca
```

如果存在 `doca-log` 或类似条目，再尝试：

```bash
gcc hello_doca.c -o hello_doca $(pkg-config --cflags --libs doca-log)
./hello_doca
```

如果没有 pkg-config，不要硬猜路径；先查安装内容：

```bash
find /opt/mellanox/doca -name "*.pc"
find /opt/mellanox/doca -name "*doca*log*"
find /opt/mellanox/doca -name "doca_log.h"
```

## 5. 第一个 DMA 实验路线

目标：同机 local DMA copy。

前置：

```bash
/opt/mellanox/doca/tools/doca_caps --list-libs | grep -E 'dma|common'
/opt/mellanox/doca/tools/doca_caps | grep -i dma -A 50
```

实验步骤：

1. 打开支持 DMA memcpy 的设备；
2. 创建 `doca_dma`；
3. `doca_dma_as_ctx()`；
4. 创建 `doca_pe` 并 connect ctx；
5. 配置 memcpy task callback；
6. 申请 src/dst 内存并注册 mmap；
7. 创建 src/dst `doca_buf`；
8. 提交 `doca_dma_task_memcpy`；
9. `doca_pe_progress()` 到 completion；
10. 比较 src/dst。

## 6. 第一个 RDMA 实验路线

建议先不要直接写 DOCA RDMA，先确保普通 RDMA/RoCE 环境可用：

```bash
ibdev2netdev
show_gids
rdma link
ip addr show
ping <peer-ip>
```

如果安装了 perftest：

```bash
# server
ib_write_bw

# client
ib_write_bw <server-ip-or-rdma-address>
```

DOCA RDMA 实验路线：

1. 两端能力查询；
2. 两端创建 `doca_rdma` 和 PE；
3. 建立 RDMA 连接或导出/连接；
4. 注册/导出 remote mmap；
5. 通过 Comch/socket 交换 descriptor；
6. 提交 write/read task；
7. progress completion；
8. 校验远端 buffer 内容。

## 7. 学习型目录建议

在真实机器上可建立：

```text
doca-lab/
  hello-log/
    hello_doca.c
    Makefile
  dma-local-copy/
    main.c
    README.md
  comch-hello/
    host_client.c
    dpu_server.c
    protocol.h
  rdma-write/
    initiator.c
    target.c
    protocol.h
  flow-acl/
    main.c
    rules.json
```

每个实验都记录：

- DOCA 版本；
- Host/DPU OS；
- 固件版本；
- `doca_caps` 输出摘要；
- 成功命令；
- 错误和修复方式。

## 8. 排错顺序

遇到失败时，按这个顺序排查：

1. 官方 sample 是否能跑；
2. `doca_caps --list-devs` 是否看到设备；
3. `doca_caps --list-libs` 是否有目标库；
4. 详细能力是否 supported；
5. PCI 地址是否写对；
6. Host/DPU 侧是否混淆；
7. 权限是否足够；
8. PE 是否 progress；
9. callback 是否配置；
10. 内存是否注册且生命周期正确；
11. RDMA 网络是否基础连通。

## 技术细节补充：实验环境记录模板

每个实验目录建议放一个 `ENV.md`，记录：

```markdown
# Experiment Environment

- DOCA version:
- BlueField/SuperNIC model:
- Firmware version:
- Host OS/kernel:
- DPU OS/kernel:
- PCI BDF:
- ibdev/netdev:
- representor mapping:
- MTU/RoCE mode/GID index:
- DOCA caps summary:
- Exact command:
- Result:
- Known caveats:
```

### 编译与链接细节

- 优先使用官方 sample 的 build system 或 package 提供的 pkg-config。
- `pkg-config --list-all | grep -i doca` 只能发现 pkg-config 条目；没有条目时应查 `/opt/mellanox/doca` 安装内容和官方编译说明。
- 不同 DOCA 版本库名、include path、rpath 可能变化，不要把一次实验的 flags 当成通用真理。
- Host 侧和 DPU 侧可能需要分别编译，架构、库路径、运行权限不同。

## 性能/调优视角

实验时按以下顺序建立基线：

1. 官方 Secure Channel/reference app 能跑；
2. `doca_caps` 记录目标能力；
3. Hello/log 证明链接；
4. DMA local copy 测不同块大小；
5. RDMA 先用 perftest 测网络，再跑 DOCA RDMA；
6. Flow 先 1 条规则测命中，再扩规则数；
7. 存储先测后端，再测 SNAP/Host 视角；
8. 每次只改一个参数，保存命令和结果。

推荐指标：

| 模块 | 指标 |
|---|---|
| Flow | pps、Gbps、counter hit、drop、rule insert/delete rate |
| DMA | GB/s、copy size、outstanding tasks、p99 completion latency |
| RDMA | bandwidth、message rate、latency、retry/timeout、CPU usage |
| Storage | IOPS、bandwidth、p99 latency、queue depth、backend latency |
| Control | Comch request latency、error rate、DPU ARM CPU |

## 开发中常见问题

| 问题 | 建议 |
|---|---|
| 没有真实硬件 | 只能写学习文档/接口草图；不要声称验证了 offload。 |
| 示例命令失败 | 先确认 PCI 地址、权限、Host/DPU 侧、服务是否启动。 |
| 编译过不去 | 找官方 sample Makefile/CMake/pkg-config，不硬猜。 |
| 运行卡住 | 加 timeout，打印 ctx state，确认 PE progress。 |
| 结果不可复现 | 记录版本、命令、环境变量、CPU pinning、网络配置。 |
| 性能比较不公平 | warm-up、固定 CPU、相同包大小/IO size/并发，至少跑多轮。 |

## 9. 学完本章应该记住

1. 第一步永远是跑官方 reference app，而不是直接写复杂代码。
2. `doca_caps` 输出是所有 DOCA 实验的事实基线。
3. Hello DOCA 只证明链接成功，不证明硬件 offload 成功。
4. DMA/RDMA/Flow 实验都要从最小闭环开始。
5. 真实 PCI 地址、设备名、能力限制必须来自目标机器。

## 10. 下一步

继续阅读：[12. 综合设计：Comch + Flow + DMA/RDMA + Storage](12-end-to-end-rdma-flow-storage-design.md)。
