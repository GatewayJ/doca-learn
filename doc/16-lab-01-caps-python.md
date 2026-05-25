# 16. Lab 01：Capabilities Python 环境探测

本实验用 Python 包装 NVIDIA 官方 `doca_caps` 工具，形成可重复的环境探测流程。

官方参考：

- DOCA Capabilities Print Tool: <https://docs.nvidia.com/doca/sdk/DOCA-Capabilities-Print-Tool/index.html>

## 1. 实验目标

完成后你应该掌握：

1. 如何检测 `doca_caps` 是否存在；
2. 如何收集 DOCA-capable devices；
3. 如何收集 representor devices；
4. 如何收集 installed libraries；
5. 如何保存原始输出，作为后续 Flow/DMA/RDMA 实验的事实基线；
6. 为什么所有后续实验都必须先做 capability baseline。

## 2. 为什么这个实验排在第二个

在 DOCA 开发中，最常见错误之一是：**代码写完了才发现目标设备不支持目标 task**。

`doca_caps` 可以提前回答：

| 问题 | 对后续实验的影响 |
|---|---|
| 机器上有没有 DOCA-capable device | 没有设备就不能做真实 Flow/DMA/RDMA offload |
| BDF 是什么 | 后续 C 程序需要 `--pci-addr <BDF>` |
| ibdev/netdev 是什么 | RDMA/RoCE 调试要用 |
| representor 是什么 | Flow/虚拟化路径要用 |
| 哪些库 installed | DMA/RDMA/Flow/Comch 是否有 SDK 支持 |
| 哪些 task supported | 目标实验是否可跑 |
| max task/buffer 限制 | benchmark 参数上限 |

## 3. 建议目录

```text
docatest/
  labs/
    01-caps-python/
      README.md
      collect_caps.py
      output/
        raw/
        summary.json
        ENV.md
```

创建目录：

```bash
cd /home/jhw/my/docatest
mkdir -p labs/01-caps-python/output/raw
cd labs/01-caps-python
```

## 4. 前置检查与手工命令基线

在写 Python 前，先手工执行官方命令。

### 4.1 列出设备

```bash
/opt/mellanox/doca/tools/doca_caps --list-devs
```

关注：

- `PCI`；
- `ibdev_name`；
- `iface_name`；
- `pci_func_type`；
- `mac_addr`；
- `ipv4_addr` / `ipv6_addr`。

### 4.2 列出 representor

```bash
/opt/mellanox/doca/tools/doca_caps --list-rep-devs
```

关注：

- parent PCI；
- representor PCI；
- `host_index`；
- `pf_index`；
- `vf_index` / `sf_index`；
- `iface_name`；
- `pci_func_type`。

### 4.3 列出安装库

```bash
/opt/mellanox/doca/tools/doca_caps --list-libs
```

确认是否包含：

- `common`；
- `dma`；
- `rdma`；
- `flow`；
- `comm_channel` / `comch` 相关；
- `aes_gcm`、`sha`、`compress`；
- `telemetry`。

### 4.4 打印详细能力

```bash
/opt/mellanox/doca/tools/doca_caps
```

这份输出通常较长，建议原样保存。

## 5. 编写 `collect_caps.py`

文件：`labs/01-caps-python/collect_caps.py`

```python
#!/usr/bin/env python3
from __future__ import annotations

import argparse
import json
import platform
import subprocess
from datetime import datetime, timezone
from pathlib import Path


def run(cmd: list[str]) -> dict:
    p = subprocess.run(cmd, text=True, capture_output=True)
    return {
        "cmd": cmd,
        "returncode": p.returncode,
        "stdout": p.stdout,
        "stderr": p.stderr,
    }


def main() -> int:
    ap = argparse.ArgumentParser(description="Collect NVIDIA DOCA capability baseline")
    ap.add_argument("--doca-caps", default="/opt/mellanox/doca/tools/doca_caps")
    ap.add_argument("--out", default="output")
    args = ap.parse_args()

    out = Path(args.out)
    raw = out / "raw"
    raw.mkdir(parents=True, exist_ok=True)

    doca_caps = Path(args.doca_caps)
    summary = {
        "timestamp_utc": datetime.now(timezone.utc).isoformat(),
        "host": platform.node(),
        "platform": platform.platform(),
        "doca_caps": str(doca_caps),
        "doca_caps_exists": doca_caps.exists(),
        "commands": {},
    }

    commands = {
        "list_devs": [str(doca_caps), "--list-devs"],
        "list_rep_devs": [str(doca_caps), "--list-rep-devs"],
        "list_libs": [str(doca_caps), "--list-libs"],
        "all_caps": [str(doca_caps)],
        "list_loggers": [str(doca_caps), "--list-loggers"],
    }

    if not doca_caps.exists():
        print(f"ERROR: {doca_caps} does not exist")
        print("Install DOCA or pass --doca-caps <path>")
        (out / "summary.json").write_text(json.dumps(summary, indent=2, ensure_ascii=False))
        return 1

    for name, cmd in commands.items():
        result = run(cmd)
        summary["commands"][name] = {
            "cmd": cmd,
            "returncode": result["returncode"],
            "stdout_file": f"raw/{name}.stdout.txt",
            "stderr_file": f"raw/{name}.stderr.txt",
        }
        (raw / f"{name}.stdout.txt").write_text(result["stdout"])
        (raw / f"{name}.stderr.txt").write_text(result["stderr"])
        print(f"[{name}] rc={result['returncode']} stdout={len(result['stdout'])} bytes stderr={len(result['stderr'])} bytes")

    (out / "summary.json").write_text(json.dumps(summary, indent=2, ensure_ascii=False))

    env_md = out / "ENV.md"
    env_md.write_text(
        "# DOCA Capability Baseline\n\n"
        f"- Timestamp UTC: {summary['timestamp_utc']}\n"
        f"- Host: {summary['host']}\n"
        f"- Platform: {summary['platform']}\n"
        f"- doca_caps: {summary['doca_caps']}\n\n"
        "## Raw Outputs\n\n"
        "- raw/list_devs.stdout.txt\n"
        "- raw/list_rep_devs.stdout.txt\n"
        "- raw/list_libs.stdout.txt\n"
        "- raw/all_caps.stdout.txt\n"
        "- raw/list_loggers.stdout.txt\n"
    )

    print(f"Wrote {out / 'summary.json'}")
    print(f"Wrote {env_md}")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

## 6. 运行

```bash
cd /home/jhw/my/docatest/labs/01-caps-python
python3 collect_caps.py
```

如果 `doca_caps` 不在默认路径：

```bash
python3 collect_caps.py --doca-caps /path/to/doca_caps
```

预期生成：

```text
output/
  ENV.md
  summary.json
  raw/
    list_devs.stdout.txt
    list_devs.stderr.txt
    list_rep_devs.stdout.txt
    list_rep_devs.stderr.txt
    list_libs.stdout.txt
    list_libs.stderr.txt
    all_caps.stdout.txt
    all_caps.stderr.txt
    list_loggers.stdout.txt
    list_loggers.stderr.txt
```

## 7. 验证输出

### 7.1 检查设备

```bash
sed -n '1,120p' output/raw/list_devs.stdout.txt
```

至少要找到一个 `PCI:` 才能继续真实硬件实验。

### 7.2 检查库

```bash
grep -E 'common|dma|rdma|flow|comm|telemetry' output/raw/list_libs.stdout.txt
```

如果目标库不是 installed，对应 lab 先不要继续。

### 7.3 检查 detailed caps

```bash
grep -i dma -A 80 output/raw/all_caps.stdout.txt
```

也可以查 RDMA/Flow：

```bash
grep -i rdma -A 80 output/raw/all_caps.stdout.txt
grep -i flow -A 80 output/raw/all_caps.stdout.txt
```

## 8. 结果如何用于后续实验

| 后续实验 | 使用哪些输出 |
|---|---|
| Hello Log | 主要不依赖设备，但可记录 DOCA 环境 |
| DMA local copy | `list_devs` 选择 BDF，`all_caps` 查 dma memcpy support |
| Comch | `list_devs` 选择 Host/DPU 侧设备，确认 comm/comch 相关库 |
| RDMA write/read | `list_devs` 获取 ibdev/netdev，`all_caps` 查 rdma task support |
| Flow ACL | `list_devs` + `list_rep_devs` 选择 port/representor/uplink，查 flow support |
| 性能压测 | 从 `all_caps` 提取 max task、buffer、queue 限制 |

## 9. 可选增强：生成简短摘要

初期先保存 raw 输出即可。后续可以逐步增加解析逻辑：

- 从 `list_devs` 提取 PCI/ibdev/netdev；
- 从 `list_libs` 提取 installed libraries；
- 从 `all_caps` 提取 supported/unsupported；
- 输出 `devices.csv`、`libs.csv`、`capabilities.csv`。

不要一开始写过度复杂 parser，因为 `doca_caps` 输出格式可能随版本变化。

## 10. 常见问题

| 问题 | 原因 | 处理 |
|---|---|---|
| `doca_caps` 不存在 | 未安装 DOCA 或路径不同 | 查安装路径，或传 `--doca-caps` |
| `--list-devs` 为空 | 驱动/固件/权限/设备模式问题 | 检查是否在正确 Host/DPU 侧，查 `lspci`/驱动 |
| `--list-libs` 没有 dma/rdma/flow | 对应库未安装或版本不含 | 安装对应 DOCA package，或换实验 |
| raw 输出很大 | detailed caps 本来就多 | 保留 raw，后续按需 grep/解析 |
| Python 脚本成功但硬件实验失败 | caps 只是基线，不替代每个模块 API capability query | C 程序里仍要做 capability query |

## 11. 通过标准

本实验通过条件：

1. `collect_caps.py` 能检测 `doca_caps` 是否存在；
2. 能生成 `output/summary.json`；
3. 能保存 `--list-devs`、`--list-rep-devs`、`--list-libs`、完整 caps 和 loggers 的 raw 输出；
4. `output/ENV.md` 记录了主机、平台和输出文件；
5. 后续实验能从 raw 输出中选择 BDF、ibdev/netdev、representor 和目标 library。

## 12. 下一步

继续阅读：[17. Lab 02：DMA Local Copy（C）](17-lab-02-dma-local-copy-c.md)。
