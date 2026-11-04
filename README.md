# UniXNU-luming2-test

Xiaomi Civi 5 Pro（`luming` / SM8735）运行 iOS 17.7.11（21H461）的**移植进度记录**。

> 这是进度与证据摘要仓库，**不包含** Apple 固件、IPSW、kernelcache、trustcache、userdata 镜像或任何 Apple 二进制/反汇编片段。
> 不提供刷机镜像下载，不保证可用，仅供技术记录。

## 当前状态（2026-11-04）

| 能力 | 状态 |
|---|---|
| Boot chain：ABL → BootShim/UEFI → DarwinBoot → XNU | ✅ |
| UFS / APFS 根分区挂载、root hash 认证、trust cache | ✅ |
| IOMobileGraphicsFamily + Legacy CPU display provider | ✅ 可出画（Apple logo） |
| 真实帧复制 / retire | ✅ 观测到 3 帧（1236×2750×4） |
| SpringBoard / backboardd / launchd / lsd 存活 | ✅ 无 panic 退出 |
| iOS 桌面 / 锁屏 | ❌ 卡在用户态 Mach IPC 等待 |
| GPU / Metal（Adreno 740） | ❌ 未实现 |
| 触摸 / IOHID | ❌ 未实现 |
| USB（设备模式 / 网络 / syslog） | ❌ 未实现 |
| 声音 / 电池 / 充电 / 传感器 | ❌ 未实现 |

## 已确认的关键结论

1. **显示链路是通的**：provider 完成 validate → copy → retire，整帧 13,596,000 字节，采样源/目标一致，`blank=0`。
2. **不是 FBSDisplayMode / CADisplay 卡住**：Stage12–13 已越过 `FBSDisplayMode` 参数断言；`refresh publish` 与 2× UI scale 均回读成功。
3. **不是完成握手（selector 6 / completion ID）卡住**：Stage18 记录到完成 ID 0→1→2 正常推进；`mode 0` 返回 Busy 符合原版非阻塞逻辑。
4. **真正的阻塞点在用户态 Mach IPC**：
   - SpringBoard 线程阻塞在 `ipc_mqueue_receive_continue`，用户栈落在 Foundation / libxpc（同步 XPC 等回复）。
   - `lsd`（launchservicesd）worker 同样阻塞在 Mach IPC 收消息，调用路径经过 TelephonyUtilities / FMIPCore 相关逻辑。
5. **平台服务缺口**：内核 IORegistry 中未观察到 `IOWatchdog` / `AppleKeyStore` 节点；`liveactivitiesd` 因 Keybag 事件注册失败退出，`watchdogd` 因查找 IOWatchdog 服务失败退出。这些是**独立确认的问题**，与桌面停帧的因果关系仍待端口归属证据。

## 阶段进展

| 阶段 | 内容 | 结果 |
|---|---|---|
| Stage 9 | ADT `AAPL,phandle` 修复 + auth blob | UFS/APFS 根挂载成功 |
| Stage 10 | Legacy CPU display provider（358 vtable, 0x18a0 ABI） | 首次出现图形帧 |
| Stage 11 | 短日志 / 崩溃摘要（CDX11） | 能稳定读到 SpringBoard 崩溃地址 |
| Stage 12 | FBSDisplayMode 参数修复（60Hz 发布） | 越过 mode 断言 |
| Stage 13 | 2× UI scale 发布（`luming-display-ui-scale=2`） | **Apple logo 出现** |
| Stage 14 / 14.1 | VT 停绘 latch + panic 前恢复 console | 诊断快照可读 |
| Stage 15 | IPC 提交者 / 分阶段计数 / vsync / late 采样 | 确认 backboardd 连接、3 帧、无 vsync 订阅 |
| Stage 16–17 | vsync 状态 / selector 统计 | 排除定时器分配失败；得到真实 selector 错误 |
| Stage 18 | wait/completion 配对 | 完成 ID 正常推进；Busy 合法 |
| Stage 19 / 19.x | stackshot 用户态栈 | 定位 SpringBoard 同步 XPC 等待 |
| Stage 20 | 多页 kcdata 解析（backboardd/SpringBoard/launchd/lsd） | 定位 lsd worker 的 Mach IPC 等待 |
| Stage 21 | 调用者 LR / owner batch | 捕获 `__posix_spawn`、CoreFoundation、BaseBoard 调用链 |

## 复现与构建

- CI 仓库：[`infinityEX-RSXQ/UniXNU-luming-f2-ci`](https://github.com/infinityEX-RSXQ/UniXNU-luming-f2-ci)（公开）
- 源码固定版本：
  - UniXNU core `33da81310819b86d3ea62cb704f156c56407764a`
  - XNU `d8b80295118ef25ac3a784134bcf95cd8e88109f`
  - 原机 kernelcache SHA256 `f18b4df865405f2a287b84977d7aca745e056d656d9fc9df1662970ce2969de7`（21H461）
- 构建/验证步骤见 [docs/BUILD.md](docs/BUILD.md)。

## 边界

- 不提供任何 Apple 受版权保护的二进制或反汇编内容；本仓库只记录结论、指标与源码级诊断思路。
- 所有“已确认”均来自真机照片/日志；未取得的字段一律标注 `unavailable`，不做推断。
- 禁止把本仓库内容当作可发布/可日常使用的 iOS 方案。

## License

本仓库文档采用 MIT。设备侧源码与补丁衍生于 UniXNU（BSD-2-Clause-Patent），以其许可证为准；Apple 相关代码与二进制不在本仓库内。
