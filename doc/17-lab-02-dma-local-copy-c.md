# 17. Lab 02：DMA Local Copy（C）

本实验进入第一个真实 DOCA Core/task 模型实验：使用 DOCA DMA 做本地 memory copy。

官方参考：

- DOCA DMA: <https://docs.nvidia.com/doca/sdk/DOCA-DMA/index.html>
- DOCA DMA Copy Application Guide: <https://docs.nvidia.com/doca/sdk/DOCA-DMA-Copy-Application-Guide/index.html>

官方文档中示例 sample 构建方式包含：

```bash
cd /opt/mellanox/doca/samples/doca_dma/dma_local_copy
meson /tmp/build
ninja -C /tmp/build
```

应用级 DMA Copy 参考应用构建方式包含：

```bash
cd /opt/mellanox/doca/applications/
meson /tmp/build -Denable_all_applications=false -Denable_dma_copy=true
ninja -C /tmp/build
```

本章不复制官方完整 sample，而是定义你自己写 lab 时应遵循的步骤和代码结构。

## 1. 实验目标

完成后你应该掌握：

1. 如何选择支持 DMA memcpy task 的 device；
2. 如何创建 `doca_dma` 并转为 `doca_ctx`；
3. 如何创建 `doca_pe` 并连接 ctx；
4. 如何注册 src/dst memory 到 `doca_mmap`；
5. 如何创建 `doca_buf_inventory` 和 `doca_buf`；
6. 如何配置 memcpy task callback；
7. 如何提交 `doca_dma_task_memcpy`；
8. 如何 `doca_pe_progress()` 到 completion；
9. 如何比较 src/dst 数据；
10. 如何按反向顺序 cleanup。

## 2. 实验边界

| 项目 | 说明 |
|---|---|
| 是否需要 DOCA-capable device | 需要 |
| 是否需要 RDMA 网络 | 不需要 |
| 是否需要 Host↔DPU 双端 | 不需要，先做 local copy |
| 是否验证硬件 DMA 引擎 | 取决于设备和 capability，需以 `doca_caps` 和官方 sample 为准 |
| 是否适合性能结论 | 初次实验不下结论，只验证功能 |

## 3. 建议目录

```text
docatest/
  labs/
    02-dma-local-copy-c/
      README.md
      main.c
      Makefile
      ENV.md
```

创建目录：

```bash
cd /home/jhw/my/docatest
mkdir -p labs/02-dma-local-copy-c
cd labs/02-dma-local-copy-c
```

## 4. 前置检查

先运行 Lab 01：

```bash
cd /home/jhw/my/docatest/labs/01-caps-python
python3 collect_caps.py
```

然后确认：

```bash
/opt/mellanox/doca/tools/doca_caps --list-devs
/opt/mellanox/doca/tools/doca_caps --list-libs | grep -i dma
/opt/mellanox/doca/tools/doca_caps | grep -i dma -A 80
```

你需要记录：

- 选择的 PCI BDF，例如 `<BDF>`；
- `dma` library 是否 installed；
- memcpy task 是否 supported；
- max task 数；
- max buffer size；
- max buffer list length。

## 5. 推荐先跑官方 sample

在写自己的代码前，先跑官方 sample：

```bash
cd /opt/mellanox/doca/samples/doca_dma/dma_local_copy
meson /tmp/doca_dma_local_build
ninja -C /tmp/doca_dma_local_build
/tmp/doca_dma_local_build/<sample_binary> -h
```

`<sample_binary>` 名称以实际构建输出为准。官方文档建议可用：

```bash
/tmp/build/<sample_name> -h
```

目的：

- 确认官方样例在你的机器上可以构建；
- 查看真实命令行参数；
- 复制可靠的编译/链接方式；
- 避免先写自己的代码再排环境问题。

## 6. 自写 lab 的代码结构

`main.c` 推荐结构：

```text
main
  parse_args
  init_log
  allocate_src_dst
  fill_src_pattern
  find_dma_capable_device
  open_doca_dev
  create_doca_dma
  create_doca_pe
  configure_dma_task_callbacks
  connect_ctx_to_pe
  create_mmaps_for_src_dst
  create_buf_inventory
  create_src_dst_doca_buf
  start_ctx
  allocate_dma_memcpy_task
  submit_task
  progress_until_done_or_timeout
  verify_dst_equals_src
  cleanup_reverse_order
```

## 7. 关键 API/对象清单

| 对象/API | 作用 |
|---|---|
| `doca_devinfo` / `doca_dev` | 查找并打开目标设备 |
| DMA capability API | 判断 memcpy task 是否 supported，读取 max 限制 |
| `doca_dma` | DMA 模块对象 |
| `doca_dma_as_ctx()` | 转为通用 `doca_ctx` |
| `doca_pe` | Progress Engine |
| `doca_dma_task_memcpy_set_conf()` | 配置 memcpy task callback/max tasks |
| `doca_mmap` | 注册 src/dst memory |
| `doca_buf_inventory` | 创建 buffer descriptor 池 |
| `doca_buf` | 描述 src/dst buffer |
| `doca_dma_task_memcpy` | 一次 memcpy task |
| `doca_pe_progress()` | 推进 completion |

具体函数签名请以安装版本头文件为准。

## 8. Completion 设计

定义一个请求状态：

```c
struct dma_copy_state {
    bool done;
    bool success;
    doca_error_t result;
};
```

callback 只做轻量事情：

```text
success callback:
  state->done = true
  state->success = true
  state->result = DOCA_SUCCESS

error callback:
  state->done = true
  state->success = false
  state->result = task_result
```

不要在 callback 中做：

- 大量日志；
- 阻塞 IO；
- 复杂锁；
- 释放仍被其他路径访问的对象。

## 9. Progress loop

示意：

```c
while (!state.done && !timeout) {
    doca_pe_progress(pe);
}
```

实际代码要加入：

- timeout；
- progress 返回值处理；
- signal handling；
- error callback；
- ctx state 检查。

## 10. 数据校验

src 填充固定 pattern：

```text
src[i] = i & 0xff
```

DMA 完成后比较：

```text
memcmp(src, dst, size) == 0
```

建议测试 size：

```text
64B, 4K, 64K, 1M, 16M
```

初次实验只跑 4K 或 1M，确认功能后再 sweep。

## 11. 构建

优先参考官方 sample 的 Meson 构建和编译参数。

如果使用 Makefile，先查 pkg-config：

```bash
pkg-config --list-all | grep -i doca
```

不要硬猜 DMA 库名。若存在合适条目，可让 Makefile 支持覆盖：

```makefile
CC ?= gcc
CFLAGS ?= -O2 -g -Wall -Wextra
PKGS ?= <按实际 pkg-config 名称填写>
CFLAGS += $(shell pkg-config --cflags $(PKGS))
LDLIBS += $(shell pkg-config --libs $(PKGS))

all: dma_local_copy

dma_local_copy: main.c
	$(CC) $(CFLAGS) $< -o $@ $(LDLIBS)

clean:
	rm -f dma_local_copy
```

如果没有 pkg-config，直接复用官方 sample 的构建系统更安全。

## 12. 运行

示例命令：

```bash
./dma_local_copy --pci-addr <BDF> --size 1048576
```

参数建议：

| 参数 | 说明 |
|---|---|
| `--pci-addr <BDF>` | 从 `doca_caps --list-devs` 选择 |
| `--size <bytes>` | copy 大小，先用 1M |
| `--repeat <N>` | 可选，后续 benchmark 用 |
| `--json <path>` | 可选，记录结果 |

## 13. 预期输出

建议输出：

```text
selected device: <BDF>
dma memcpy supported: yes
max tasks: <N>
max buf size: <bytes>
copy size: 1048576
submit task: ok
completion: success
verify: PASS
```

## 14. 性能扩展

功能通过后再测性能：

```bash
for s in 64 4096 65536 1048576 16777216; do
  ./dma_local_copy --pci-addr <BDF> --size $s --repeat 1000
done
```

注意：

- 小块 copy 可能比 CPU memcpy 慢；
- 注册内存不要放在 hot path；
- 多 outstanding task 才可能填满 DMA engine；
- 测性能时 C binary 内部计时，Python 只调度。

## 15. 常见问题

| 现象 | 可能原因 | 处理 |
|---|---|---|
| `unsupported` | 设备不支持 DMA memcpy task | 换设备/侧，查 `doca_caps` 和 capability API |
| start ctx 失败 | mandatory config 未完成 | 检查 device、callback、max task、PE connect |
| task 不完成 | 没有 progress PE 或 callback 未配置 | 加 timeout 和状态日志 |
| data mismatch | buffer length、offset、cache/coherency、生命周期错误 | 简化到 4K，固定 pattern，逐步检查 |
| 性能低 | size 太小、注册在 hot path、单 outstanding、CPU 抢占 | 复用 mmap，增加 batch/QD，pin CPU |
| cleanup 崩溃 | task 未完成就释放 mmap/buf | completion 后再反向释放 |

## 16. 通过标准

本实验通过条件：

1. 能选择目标 BDF；
2. 能打印 DMA capability 摘要；
3. 能成功提交 memcpy task；
4. 能收到 success callback；
5. `memcmp(src, dst)` 通过；
6. 错误路径不会泄露或崩溃；
7. `ENV.md` 记录了 DOCA 版本和 `doca_caps` 摘要。

## 17. 下一步

继续阅读：[18. Lab 03：Comch Host/DPU 控制通道（C）](18-lab-03-comch-host-dpu-c.md)。
