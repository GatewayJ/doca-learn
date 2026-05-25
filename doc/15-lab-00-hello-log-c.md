# 15. Lab 00：Hello DOCA Log（C）

本实验是 DOCA 代码实验的第一个最小闭环：**验证 C 编译环境、DOCA 头文件、DOCA Log 库链接和运行时日志输出**。

它不验证 DPU offload，不验证 Flow/DMA/RDMA，只验证“我能编译并运行一个最小 DOCA C 程序”。

官方参考：

- DOCA Log: <https://docs.nvidia.com/doca/sdk/DOCA-Log/index.html>
- DOCA Developer Quick Start Guide: <https://docs.nvidia.com/doca/sdk/DOCA-Developer-Quick-Start-Guide/index.html>

## 1. 实验目标

完成后你应该掌握：

1. 如何创建一个最小 C lab 目录；
2. 如何查找 DOCA 头文件、库和 pkg-config 信息；
3. 如何使用 `doca_log_backend_create_standard()`；
4. 如何使用 `DOCA_LOG_REGISTER()` 和 `DOCA_LOG_INFO()`；
5. 如何区分“链接成功”和“硬件 offload 成功”。

## 2. 实验边界

| 项目 | 说明 |
|---|---|
| 是否需要 BlueField/DPU | 不一定；只要安装了 DOCA SDK/headers/libs 即可尝试 |
| 是否需要真实 DOCA device | 不需要 |
| 是否证明硬件 offload | 不证明 |
| 是否涉及 `doca_dev` | 不涉及 |
| 是否涉及 `doca_pe` | 不涉及 |
| 主要验证对象 | C 编译、DOCA Log 头文件、链接、运行时日志 |

## 3. 建议目录

```text
docatest/
  labs/
    00-hello-log-c/
      README.md
      hello_doca.c
      Makefile
      ENV.md
```

创建目录：

```bash
cd /home/jhw/my/docatest
mkdir -p labs/00-hello-log-c
cd labs/00-hello-log-c
```

## 4. 前置检查

### 4.1 检查 DOCA 安装路径

官方示例中 DOCA 常见安装路径为：

```bash
/opt/mellanox/doca/
```

先检查：

```bash
test -d /opt/mellanox/doca && echo "DOCA path exists" || echo "DOCA path missing"
```

### 4.2 查找 Log 头文件

```bash
find /opt/mellanox/doca -name "doca_log.h" 2>/dev/null
```

如果找不到：

- 可能没有安装开发包；
- 可能路径不是 `/opt/mellanox/doca`；
- 可能在容器中没有挂载 DOCA SDK；
- 需要回到官方安装文档确认安装类型。

### 4.3 检查 pkg-config

```bash
pkg-config --list-all | grep -i doca
```

如果看到类似 `doca-log` 或相关条目，可以优先用 pkg-config 构建。

> 注意：不同 DOCA 版本的 pkg-config 名称可能不同。不要把某台机器上的名字硬写成通用规则。

## 5. 编写 `hello_doca.c`

文件：`labs/00-hello-log-c/hello_doca.c`

```c
#include <stdio.h>
#include <doca_log.h>

DOCA_LOG_REGISTER(HELLO_DOCA);

int main(int argc, char **argv)
{
    (void)argc;
    (void)argv;

    doca_log_backend_create_standard();

    DOCA_LOG_INFO("Hello, DOCA Log!");
    printf("Hello from normal printf too.\n");

    return 0;
}
```

代码说明：

| 代码 | 作用 |
|---|---|
| `#include <doca_log.h>` | 引入 DOCA Log API |
| `DOCA_LOG_REGISTER(HELLO_DOCA)` | 注册当前文件/模块 logger |
| `doca_log_backend_create_standard()` | 创建标准日志后端，通常输出到 stderr/stdout |
| `DOCA_LOG_INFO(...)` | 输出 INFO 级 DOCA 应用日志 |
| `printf(...)` | 普通 C 输出，便于区分 DOCA log 和普通输出 |

## 6. Makefile 方案

### 6.1 pkg-config 方案

如果 `pkg-config --list-all | grep -i doca` 能找到 log 相关条目，可以写：

```makefile
CC ?= gcc
PKG ?= doca-log
CFLAGS ?= -O2 -g -Wall -Wextra

CFLAGS += $(shell pkg-config --cflags $(PKG))
LDLIBS += $(shell pkg-config --libs $(PKG))

all: hello_doca

hello_doca: hello_doca.c
	$(CC) $(CFLAGS) $< -o $@ $(LDLIBS)

clean:
	rm -f hello_doca
```

如果你的机器上 pkg-config 名称不是 `doca-log`，需要先查真实名称：

```bash
pkg-config --list-all | grep -i 'doca.*log\|log.*doca'
```

然后构建时覆盖：

```bash
make PKG=<真实 pkg-config 名称>
```

### 6.2 没有 pkg-config 时怎么办

不要直接猜 include/lib 路径。先探索：

```bash
find /opt/mellanox/doca -name "*.pc" 2>/dev/null
find /opt/mellanox/doca -name "doca_log.h" 2>/dev/null
find /opt/mellanox/doca -name "*doca*log*" 2>/dev/null
```

然后参考官方 sample 或安装包文档确定编译参数。

## 7. 构建

```bash
cd /home/jhw/my/docatest/labs/00-hello-log-c
make
```

预期：生成 `hello_doca` 二进制。

如果失败：

| 错误 | 可能原因 | 处理 |
|---|---|---|
| `doca_log.h: No such file` | 未安装 headers 或 include path 不对 | 查 `find /opt/mellanox/doca -name doca_log.h` |
| `Package doca-log was not found` | pkg-config 名称不对或无 `.pc` 文件 | 查 `pkg-config --list-all | grep -i doca` |
| `undefined reference` | 链接库不对 | 使用官方 sample 的链接方式或真实 pkg-config |
| 运行时报 shared library not found | runtime library path 不对 | 查官方安装路径和 loader 配置，不硬写 rpath |

## 8. 运行

```bash
./hello_doca
```

预期输出包含两类信息：

```text
Hello, DOCA Log!
Hello from normal printf too.
```

实际 DOCA log 格式可能带时间戳、logger 名称、级别等，取决于 DOCA 版本和日志后端。

## 9. 验证点

| 验证点 | 判断 |
|---|---|
| 二进制能生成 | C 编译器、头文件、链接库基本可用 |
| `DOCA_LOG_INFO` 有输出 | DOCA Log 应用日志后端可用 |
| `printf` 有输出 | 程序确实运行 |
| 不需要 PCI BDF | 本实验没有打开硬件 device |

## 10. 记录 `ENV.md`

建议写：

```markdown
# Lab 00 ENV

- Date:
- Host OS:
- Kernel:
- DOCA path:
- pkg-config doca entries:
- doca_log.h path:
- Build command:
- Run command:
- Result:
```

可以用命令辅助收集：

```bash
uname -a
pkg-config --list-all | grep -i doca || true
find /opt/mellanox/doca -name "doca_log.h" 2>/dev/null
```

## 11. 常见问题

| 现象 | 可能原因 | 处理 |
|---|---|---|
| 找不到 `doca_log.h` | 开发头文件未安装或路径不同 | 查 `find /opt/mellanox/doca -name doca_log.h`，确认安装开发包 |
| pkg-config 找不到 DOCA | 没有 `.pc` 文件或名称不同 | `pkg-config --list-all | grep -i doca`，或复用官方 sample 构建方式 |
| 链接时报 undefined reference | 链接库不完整 | 用真实 pkg-config 条目或官方 sample flags |
| 运行时报 shared library not found | runtime library path 不在 loader 搜索路径 | 按官方安装说明配置 loader，不随意硬写 rpath |
| DOCA log 没输出但 printf 有输出 | 日志后端/日志级别问题 | 确认 `doca_log_backend_create_standard()` 和 log level |

## 12. 通过标准

本实验通过条件：

1. `hello_doca.c` 能编译；
2. 二进制能运行；
3. `DOCA_LOG_INFO` 和 `printf` 都有输出；
4. `ENV.md` 记录了 DOCA Log 头文件路径和构建命令；
5. 文档中明确说明本实验不证明硬件 offload。

## 13. 常见误解

### 13.1 Hello DOCA 成功不代表有 DPU

本实验只用 DOCA Log，不需要打开设备。因此它只能说明：

- 编译器可用；
- DOCA header 可用；
- DOCA log library 可链接；
- 程序可运行。

它不能说明：

- `doca_dev` 可打开；
- Flow/DMA/RDMA 支持；
- BlueField/SuperNIC 能力存在；
- 硬件 offload 生效。

### 13.2 不要把编译参数写死

DOCA 版本、OS、安装方式不同，include/lib/pkg-config 都可能不同。实验 README 应记录“本机实际参数”，不要把它当作所有机器通用配置。

## 12. 下一步

继续阅读：[16. Lab 01：Capabilities Python 环境探测](16-lab-01-caps-python.md)。
