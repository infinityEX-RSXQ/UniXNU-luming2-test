# 构建与验证

本仓库不提供镜像，只记录构建流程。需要一台 macOS（推荐 Apple Silicon，Xcode 16.4）或 GitHub macOS runner。

## 固定输入

| 项 | 值 |
|---|---|
| UniXNU core | `33da81310819b86d3ea62cb704f156c56407764a` |
| XNU | `d8b80295118ef25ac3a784134bcf95cd8e88109f` |
| 原机 kernelcache | `f18b4df865405f2a287b84977d7aca745e056d656d9fc9df1662970ce2969de7`（iPadOS 17.7.11 / 21H461） |
| defconfig | `luming_defconfig` |

## CI

公开 CI 仓库：[`infinityEX-RSXQ/UniXNU-luming-f2-ci`](https://github.com/infinityEX-RSXQ/UniXNU-luming-f2-ci)

`workflow_dispatch` 主要输入：

```
unixnu_repository=UniXNU/UniXNU
unixnu_ref=33da81310819b86d3ea62cb704f156c56407764a
device_ref=<设备分支>
defconfig=luming_defconfig
steps=kernel
xcode=16.4
kernelcache_sha256=f18b4df865405f2a287b84977d7aca745e056d656d9fc9df1662970ce2969de7
```

`steps=kernel` 产出 `kernel.macho`、`build-manifest.json`、`build-report.json`、ABI 头、各阶段安装/检查报告。

## 产物校验

1. `kernel.macho` SHA256 与 `build-manifest.json` 一致；
2. `build-report.json` 中 `virtual_entries=358`、`generated_abi_sha256` 与基线一致、`original_instructions_modified=false`；
3. 各 stage 的 `*-kernel-check.json` 全绿；
4. 负向对照：旧 stage 内核应被当前 qualifier 拒绝（缺少新符号）。

## super 打包

将新 `kernel.macho` 替换进已验证的 super（保留 ADT、auth blob、trust cache、Data seed 与 control boot），并在参数中追加对应 stage 的 boot-arg。本仓库不包含这些镜像。

## 真机验证

- `fastboot flash super` + `fastboot boot`（仅 RAM boot，不写 bootloader/GPT）
- 15 秒后主动快照/panic，读取屏幕短行摘要
- 未取得的字段必须报告 `unavailable`，不得推断
