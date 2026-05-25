# 18. Lab 03：Comch Host/DPU 控制通道（C）

本实验开始构建 Host↔DPU 成对程序：Host 侧 client 发送控制消息，DPU 侧 server 接收并返回响应。

官方参考：

- DOCA Comch: <https://docs.nvidia.com/doca/sdk/DOCA-Comch/index.html>

官方文档中 sample 构建方式包含：

```bash
cd /opt/mellanox/doca/samples/doca_comch/<sample_name>
meson /tmp/build
ninja -C /tmp/build
/tmp/build/<sample_name> -h
```

## 1. 实验目标

完成后你应该掌握：

1. Comch server/client 的基本角色；
2. Host 和 DPU 两端程序如何分工；
3. 如何设计最小控制协议；
4. 如何处理连接建立、消息发送、响应、断开；
5. 为什么 Comch 只适合控制面/元数据，不适合搬大数据；
6. 后续如何用 Comch 控制 Flow/DMA/RDMA 资源。

## 2. 实验边界

| 项目 | 说明 |
|---|---|
| 是否需要 Host+DPU 双端 | 推荐需要 |
| 是否搬大块数据 | 不搬，只发小消息 |
| 是否涉及 Flow/DMA/RDMA | 不直接涉及，只为后续做控制面基础 |
| 是否可用 socket 替代 | 可以，但本实验学习 DOCA Comch |
| 是否验证硬件 fast path | 不验证 |

## 3. 建议目录

```text
docatest/
  labs/
    03-comch-host-dpu-c/
      README.md
      protocol.h
      host_client.c
      dpu_server.c
      Makefile
      ENV.md
```

创建目录：

```bash
cd /home/jhw/my/docatest
mkdir -p labs/03-comch-host-dpu-c
cd labs/03-comch-host-dpu-c
```

## 4. 前置检查

在 Host 和 DPU 侧分别确认：

```bash
/opt/mellanox/doca/tools/doca_caps --list-devs
/opt/mellanox/doca/tools/doca_caps --list-libs | grep -Ei 'comm|comch|cc'
/opt/mellanox/doca/tools/doca_caps | grep -Ei 'comm|comch' -A 80
```

记录：

- Host 侧可用 BDF；
- DPU 侧可用 BDF；
- Comch/comm_channel 相关库是否 installed；
- service name；
- 两端 DOCA 版本。

## 5. 推荐先跑官方 sample

```bash
cd /opt/mellanox/doca/samples/doca_comch/<sample_name>
meson /tmp/doca_comch_build
ninja -C /tmp/doca_comch_build
/tmp/doca_comch_build/<sample_name> -h
```

先用官方 sample 证明：

- Host 能连接 DPU；
- 两侧 service/device 参数正确；
- 权限和运行环境没问题。

## 6. 最小协议设计

文件：`protocol.h`

```c
#pragma once

#include <stdint.h>

#define DOCA_LAB_MAGIC 0x444f4341u /* 'DOCA' */
#define DOCA_LAB_VERSION 1

enum lab_opcode {
    LAB_OP_HELLO = 1,
    LAB_OP_ECHO = 2,
    LAB_OP_QUERY_CAPS = 3,
    LAB_OP_BYE = 4,
    LAB_OP_ERROR = 255,
};

struct lab_msg_header {
    uint32_t magic;
    uint16_t version;
    uint16_t opcode;
    uint64_t request_id;
    uint32_t payload_len;
    uint32_t status;
};
```

设计原则：

| 字段 | 作用 |
|---|---|
| `magic` | 防止误解析 |
| `version` | Host/DPU 协议版本协商 |
| `opcode` | 消息类型 |
| `request_id` | 日志和响应关联 |
| `payload_len` | 边界检查 |
| `status` | 响应状态或错误码 |

## 7. Server 侧流程

`dpu_server.c` 结构：

```text
main
  parse_args(service_name, pci_addr)
  init_log
  open_doca_dev
  create_pe
  create_comch_server
  configure_connection_callback
  configure_receive_callback
  connect_ctx_to_pe
  start_ctx
  progress_loop
    on client connected
    on message received
      validate header
      handle HELLO/ECHO/QUERY_CAPS/BYE
      submit response send task
    on client disconnected
      cleanup per-connection resources
  stop_ctx
  cleanup
```

Server 必须做的校验：

1. `magic` 是否正确；
2. `version` 是否支持；
3. `payload_len` 是否超过最大消息；
4. `opcode` 是否在白名单；
5. `request_id` 是否重复或乱序；
6. 连接断开时是否释放资源。

## 8. Client 侧流程

`host_client.c` 结构：

```text
main
  parse_args(service_name, pci_addr)
  init_log
  open_doca_dev
  create_pe
  create_comch_client
  configure_connection_callback
  configure_receive_callback
  connect_ctx_to_pe
  start_ctx
  wait_connected
  send HELLO
  wait response
  send ECHO
  wait response
  send QUERY_CAPS
  wait response
  send BYE
  stop_ctx
  cleanup
```

Client 输出建议：

```text
connect: success
send HELLO request_id=1
recv HELLO response status=0
send ECHO request_id=2 payload="hello"
recv ECHO response payload="hello"
send BYE request_id=3
complete
```

## 9. Comch task/event 注意事项

DOCA Comch 文档中包含 server/client/consumer/producer 初始化流程、control channel send task、receive event、connection status changed event 等内容。

学习阶段先关注：

| 项 | 说明 |
|---|---|
| server initialization | DPU 侧创建 server、设置 service name 和 callbacks |
| client initialization | Host 侧创建 client、连接 server |
| connection event | server 感知 client 连接/断开 |
| receive event | 收到控制消息 |
| send task | 发送响应或请求 |
| PE progress | 推进连接和消息事件 |

不要一开始就引入 producer/consumer/DPA MsgQ，先跑通普通控制消息。

## 10. 构建

建议 Host/DPU 两侧分别构建。架构、路径和 pkg-config 可能不同。

```bash
# Host side
make host_client

# DPU side
make dpu_server
```

Makefile 不要硬编码不可验证的库名。优先参考官方 Comch sample 的 Meson 构建参数。

如果用 Makefile，建议支持覆盖：

```makefile
CC ?= gcc
CFLAGS ?= -O2 -g -Wall -Wextra
PKGS ?= <按实际 pkg-config 名称填写>
CFLAGS += $(shell pkg-config --cflags $(PKGS))
LDLIBS += $(shell pkg-config --libs $(PKGS))

all: host_client dpu_server

host_client: host_client.c protocol.h
	$(CC) $(CFLAGS) host_client.c -o $@ $(LDLIBS)

dpu_server: dpu_server.c protocol.h
	$(CC) $(CFLAGS) dpu_server.c -o $@ $(LDLIBS)

clean:
	rm -f host_client dpu_server
```

## 11. 运行步骤

### 11.1 DPU 侧启动 server

```bash
./dpu_server --pci-addr <DPU_BDF> --service-name doca_lab_comch
```

### 11.2 Host 侧启动 client

```bash
./host_client --pci-addr <HOST_BDF> --service-name doca_lab_comch
```

### 11.3 预期输出与验证

Server 侧应看到：

```text
server started service=doca_lab_comch
client connected
recv HELLO request_id=1
send HELLO response
recv ECHO request_id=2
send ECHO response
client disconnected
```

Client 侧应看到：

```text
connected
HELLO: OK
ECHO: OK
BYE: OK
```

## 12. 调优与设计扩展

Comch 是控制面，调优重点不是吞吐，而是稳定性：

- request_id；
- timeout；
- heartbeat；
- payload length limit；
- outstanding request limit；
- batch；
- 断连 cleanup；
- version negotiation。

后续可以增加：

| 扩展 | 目的 |
|---|---|
| `CREATE_FLOW` opcode | Host 控制 DPU 下发 Flow rule |
| `REGISTER_MEMORY` opcode | 交换 DMA/RDMA memory metadata |
| `RDMA_CONNECT` opcode | 交换 RDMA 连接信息 |
| `QUERY_STATS` opcode | 查询 Flow counter/telemetry |
| heartbeat | 检测连接存活 |
| auth token | 控制面简单鉴权 |

## 13. 常见问题

| 现象 | 可能原因 | 处理 |
|---|---|---|
| client 连不上 | server 未启动、service name 错、BDF 错 | 先跑官方 sample，确认两端参数 |
| server 收到乱码 | 协议 header/大小端/结构体 padding 问题 | 固定 header，使用明确整数类型，必要时网络序列化 |
| 收到消息但无响应 | send task 未提交或 PE 未 progress | 检查 callback 和 progress loop |
| 断开后资源残留 | connection cleanup 未实现 | 每个 connection 建 resource list |
| 控制面卡住 | callback 做了重活或锁死 | callback 只入队，业务线程处理 |
| 大消息很慢 | Comch 不适合 bulk data | 大数据改走 DMA/RDMA |

## 14. 通过标准

本实验通过条件：

1. DPU server 能启动并等待连接；
2. Host client 能连接；
3. HELLO/ECHO/BYE 至少三类消息正常；
4. request_id 能在两侧日志对应；
5. 错误 header 会被拒绝；
6. client 断开后 server 清理连接状态；
7. `ENV.md` 记录 Host/DPU 两侧 BDF 和 DOCA 版本。

## 15. 下一步

继续阅读：[19. Lab 04：RDMA Write/Read（C）](19-lab-04-rdma-write-read-c.md)。
