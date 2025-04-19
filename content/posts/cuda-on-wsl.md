---
slug: cuda-on-wsl
title: 在 WSL 中启用 Cuda
date: 2025-01-16 14:15:06
---

Windows 下安装 [nVidia 驱动](https://www.nvidia.com/en-us/drivers/)

安装 WSL，这里没什么好说，按文档安装即可

> [!NOTE]
> 最好直接用 Ubuntu，大部分商业公司的软件优先支持这个发行版，使用省心

在 Windows 下配置

```toml
# ~/.wslconfig
[wsl2]
networkingMode=mirrored
```

在 WSL 中配置

```toml
# /etc/wsl.conf
[boot]
systemd=true
```

在 Windows 和 WSL 中查看 nVidia 和 CUDA 版本

```bash
nvidia-smi
```

检查 pytorch 是否能正常使用 CUDA

```python
import torch

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(device)
```
