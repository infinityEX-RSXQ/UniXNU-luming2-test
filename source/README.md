# 源码镜像（Source mirror）

本目录是 luming 移植项目所用**设备侧源码与诊断集成代码**的镜像，便于离线阅读与复现。内容与公开 CI 仓库 [`infinityEX-RSXQ/UniXNU-luming-f2-ci`](https://github.com/infinityEX-RSXQ/UniXNU-luming-f2-ci) 对应分支一致或为其拷贝。

## 内容

| 目录 | 说明 |
|---|---|
| `device/` | 设备仓库树（stage10–stage18 诊断集成、`patches/`、`drivers/`、`kits/`、`configs/`、`docs/`、`scripts/`） |
| `stage19p4/` | Stage19.4 stackshot 采集（kcdata 解析、Adapter/Model、qualifier） |
| `stage20/` | Stage20 lsd 多页/多实例解析 |
| `stage21/` | Stage21 owner 调用链采集（Format21/Relations21） |

这些目录中的 `install_stage*.py` 会把诊断代码追加/替换到构建树；`check*.py`、`qualify_*_kernel.py` 用于主机与内核校验；`*.json` 为输入 pin 与期望哈希。

## 不包含

- Apple 固件、IPSW、kernelcache、trustcache、userdata
- Apple 二进制、反汇编、类转储
- GitHub token、私有仓库凭据、本机路径
- 任何构建产物（`kernel.macho`、`super`、`boot.img`）

## 许可证

- `device/` 与各 stage 源码衍生于 UniXNU（BSD-2-Clause-Patent），以其许可证为准。
- 本仓库新增文档为 MIT。
- Apple 相关头文件/协议仅以符号名与偏移形式出现在诊断代码中，不包含 Apple 原始实现。
