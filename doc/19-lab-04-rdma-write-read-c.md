# 19. Lab 04：RDMA Write/Read（C）

本实验学习 DOCA RDMA 的最小远端内存访问流程：两端建立 RDMA 连接，注册/导出内存，执行 RDMA write/read，并校验数据。

官方参考：

- DOCA RDMA: <https://docs.nvidia.com/doca/sdk/DOCA-RDMA/index.html>
- RDMA over Converged Ethernet: <https://docs.nvidia.com/doca/sdk/RDMA-over-Converged-Ethernet/index.html>

官方文档中 sample 构建方式包含：

```bash
cd /opt/mellanox/doca/samples/doca_rdma/<sample_name>
meson /tmp/build
ninja -C /tmp/build
/tmp/build/<sample_name> -h
```

## 1. 实验目标

完成后你应该掌握：

1. RDMA 与 DMA 的差异；
2. RoCE 环境基线检查；
3. RDMA connection metadata 如何交换；
4. remote memory 为什么必须注册/导出；
5. RDMA write/read task 的基本生命周期；
6. completion 语义与应用层确认的区别；
7. RDMA 常见错误如何定位。

## 2. 实验边界

| 项目 | 说明 |
|---|---|
| 是否需要两端机器/两端环境 | 推荐需要，至少需要可通信 RDMA peer |
| 是否需要 RoCE/IB 网络 | 需要 |
| 是否需要 Comch | 不强制；可用 socket/文件/Comch 交换 metadata |
| 是否做性能结论 | 初次实验只做功能验证 |
| 是否能随便读写远端内存 | 不能，必须注册/授权/交换 descriptor |

## 3. 建议目录

```text
docatest/
  labs/
    04-rdma-write-read-c/
      README.md
      protocol.h
      initiator.c
      target.c
      Makefile
      ENV.md
```

角色：

| 程序 | 作用 |
|---|---|
| `target.c` | 注册远端 buffer，导出/提供 remote descriptor，等待读写 |
| `initiator.c` | 连接 target，拿到 descriptor，执行 write/read |
| `protocol.h` | metadata 交换协议结构 |

## 4. 前置检查：RoCE 基线检查

在写 DOCA RDMA 前，先确认普通 RDMA/RoCE 可用。

### 4.1 设备映射

```bash
ibdev2netdev
rdma link
rdma dev
ip -br addr
```

### 4.2 GID 表

官方 RoCE 文档说明可从 sysfs 查看 GID：

```bash
cat /sys/class/infiniband/<device>/ports/<port>/gids/<index>
cat /sys/class/infiniband/<device>/ports/<port>/gid_attrs/types/<index>
cat /sys/class/infiniband/<device>/ports/<port>/gid_attrs/ndevs/<index>
```

要记录：

- ibdev；
- port；
- GID index；
- RoCEv1/RoCEv2；
- 对应 netdev；
- IP；
- MTU。

### 4.3 基础连通

```bash
ping <peer-ip>
```

如果安装了 perftest，先跑：

```bash
# target side
ib_write_bw

# initiator side
ib_write_bw <target-ip-or-host>
```

只有普通 RDMA 工具能跑通后，再进入 DOCA RDMA。

## 5. 前置 DOCA 检查

两端分别执行：

```bash
/opt/mellanox/doca/tools/doca_caps --list-devs
/opt/mellanox/doca/tools/doca_caps --list-libs | grep -i rdma
/opt/mellanox/doca/tools/doca_caps | grep -i rdma -A 120
```

记录：

- 选择的 BDF；
- RDMA library 是否 installed；
- send/recv/read/write/atomic task 支持情况；
- max tasks；
- max buffer size；
- 是否支持目标 datapath。

## 6. Metadata 交换协议

RDMA 需要交换连接和内存信息。学习阶段可以先用 TCP socket 或文件交换，后续再换 Comch。

`protocol.h` 建议：

```c
#pragma once

#include <stdint.h>

#define RDMA_LAB_MAGIC 0x52444d41u /* 'RDMA' */
#define RDMA_LAB_VERSION 1

struct rdma_lab_header {
    uint32_t magic;
    uint16_t version;
    uint16_t type;
    uint32_t payload_len;
};

struct rdma_lab_buffer_desc {
    uint64_t buffer_id;
    uint64_t length;
    uint32_t access_flags;
    /* 实际 DOCA/RDMA descriptor 不要自己发明字段；
     * 这里仅表示“需要传递一段官方 API 产生的 opaque descriptor”。
     */
};
```

重要：真实 DOCA RDMA descriptor/key/addr 格式以官方 API 为准，不要自己伪造。

## 7. Target 侧流程

```text
target main
  parse_args(pci_addr, listen_addr/port, size)
  init_log
  check_roce_env_summary
  find/open_doca_dev
  create_doca_rdma
  create_pe
  configure callbacks
  connect ctx to pe
  allocate target buffer
  create/register mmap for target buffer
  export mmap/buffer for RDMA
  start/listen/connect RDMA context
  publish descriptor through control channel
  wait for write/read operations
  validate buffer after write
  cleanup
```

Target 初始化后输出：

```text
selected device: <BDF>
rdma write supported: yes
rdma read supported: yes
target buffer size: <bytes>
listening/control endpoint: <addr>
remote descriptor ready
```

## 8. Initiator 侧流程

```text
initiator main
  parse_args(pci_addr, target_addr, size)
  init_log
  check_roce_env_summary
  find/open_doca_dev
  create_doca_rdma
  create_pe
  configure callbacks
  allocate local src/dst buffer
  register local mmap
  connect to target
  receive remote descriptor
  create remote buffer/address object
  submit RDMA write task
  progress until completion
  submit RDMA read task
  progress until completion
  verify read-back data
  cleanup
```

## 9. DOCA RDMA 关键 API/对象

| 对象/API | 作用 |
|---|---|
| `doca_rdma` | RDMA 模块对象 |
| `doca_rdma_as_ctx()` | 转成 `doca_ctx` |
| `doca_mmap_export_rdma` | 导出可被 RDMA 访问的 memory map |
| `doca_rdma_export()` / `doca_rdma_connect()` | RDMA 连接/导出相关流程，具体按官方版本使用 |
| `doca_rdma_addr_create/destroy` | 管理远端地址/连接对象 |
| task config APIs | 配置 receive/send/read/write/atomic callbacks |
| `doca_pe_progress()` | 推进 RDMA task/event completion |

函数签名和具体连接流程以安装版本官方文档为准。

## 10. 运行步骤

### 10.1 Target 侧

```bash
./rdma_target \
  --pci-addr <TARGET_BDF> \
  --listen <TARGET_IP>:<PORT> \
  --size 1048576
```

### 10.2 Initiator 侧

```bash
./rdma_initiator \
  --pci-addr <INITIATOR_BDF> \
  --target <TARGET_IP>:<PORT> \
  --size 1048576
```

### 10.3 预期输出与验证

Initiator：

```text
connect: OK
remote descriptor received
RDMA write submit: OK
RDMA write completion: OK
RDMA read submit: OK
RDMA read completion: OK
verify readback: PASS
```

Target：

```text
connection accepted
remote write observed or notified
buffer verify: PASS
connection closed
```

注意：Target 是否能“观察到 write”取决于你是否设计了应用层通知。RDMA write completion 不等于远端应用已经处理数据。

## 11. 性能扩展

功能通过后再做参数 sweep：

```text
message size: 64B, 1K, 4K, 64K, 1M
queue depth: 1, 4, 16, 64, 256
operation: write, read, send/recv
```

调优点：

- MTU；
- GID index；
- PFC/ECN/DCQCN；
- QP/CQ 配置；
- CQ moderation；
- inline size；
- MR/mmap 复用；
- NUMA 和 CPU pinning；
- queue depth。

## 12. 常见问题

| 现象 | 可能原因 | 处理 |
|---|---|---|
| perftest 不通 | RoCE/IP/GID/交换机配置问题 | 先修网络，不看 DOCA 代码 |
| DOCA RDMA connect 失败 | BDF/GID/connection metadata 错 | 打印两端设备和地址摘要 |
| remote access error | descriptor/rkey/长度/权限错误 | 检查 mmap export、remote buffer 长度和生命周期 |
| completion 不来 | PE 没 progress、QP/连接异常、网络丢包 | 加 timeout，查 RDMA counters |
| readback 不一致 | write 完成语义误解、cache 同步、buffer offset 错 | 增加应用层 ack，固定 pattern 校验 |
| 性能差 | MTU/QD/NUMA/PFC/ECN/CQ moderation 不合理 | 先对比 perftest 基线 |

## 13. 安全注意事项

- remote descriptor/key 不要写入公开日志；
- 控制通道必须校验 peer 身份；
- remote buffer 长度和权限必须最小化；
- 连接断开后撤销或释放 remote access；
- 不要让一个 tenant 拿到另一个 tenant 的 rkey/descriptor。

## 14. 通过标准

本实验通过条件：

1. 普通 RDMA/RoCE 基线可用；
2. 两端 DOCA RDMA capability 支持目标 task；
3. Initiator 能拿到 Target remote descriptor；
4. RDMA write completion 成功；
5. RDMA read completion 成功；
6. readback 数据与写入 pattern 一致；
7. 错误路径能超时退出并 cleanup。

## 15. 下一步

继续阅读：[20. Lab 05：Flow ACL + Counter（C）](20-lab-05-flow-acl-c.md)。
