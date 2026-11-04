# 详细进度（Stage 9 → Stage 21）

本文件记录每一阶段的**目标、改动、真机证据与结论**。所有数据来自真机照片/日志；未取得的字段明确标注 `unavailable`。

## Stage 9 — 根挂载修复

- 问题：设备树 owner ADT 使用 `phandle`，XNU 只注册 `AAPL,phandle`，导致 UFS 中断 parent 为 NULL，内核 data abort（pc `0xfffffff007899938`）。
- 改动：属性重命名 + 注入 `system-volume-auth-blob`（229B）。
- 结果：APFS 根分区挂载成功，root hash 认证通过。

## Stage 10 — Legacy CPU display provider

- 增加 `IOMobileGraphicsFamily` 预链接 root（原 21H461）+ 裁剪的 CPU Legacy/GOP provider。
- ABI：358 vtable slots，实例大小 `0x18a0`，`cpu_only=true`，原版指令未修改。
- 结果：首次出现图形帧（早期仅文字 console）。

## Stage 11 — 崩溃摘要（CDX11）

- 原生日志缓冲只有 96B，改为两条短事件 + panic reason 摘要。
- 定位 SpringBoard 崩溃于 FrontBoardServices `FBSDisplayMonitor`（`0x19f3df2ec`，CADisplay 为 nil）。

## Stage 12 — FBSDisplayMode 修复

- 修复 60Hz 属性发布（`luming-display-refresh-publish=1`）。
- 结果：越过 `FBSDisplayMode` 初始化断言（BRK `0x19f3d0744`）。

## Stage 13 — 2× UI scale

- `luming-display-ui-scale=2`：向 `/chosen` 发布 `display-scale=2` / `display-rotation=0`。
- 结果：**Apple logo 稳定显示**，标志显示链路真正出帧。

## Stage 14 / 14.1 — VT 停绘与诊断可见性

- VT latch：handoff 后拒绝普通 console 重接管，panic 时放行。
- 14.1：panic 前暂停 provider 写屏 → 等在途复制结束 → 恢复文字 console → 输出快照。
- 结果：CDX14 快照可读；`blank=0`，3 次真实 copy/retire，`late==last`、源/目标采样一致。

## Stage 15 — IPC/提交者/分阶段计数

- 记录 client owner（`backboardd` pid 58 等）、submit/validate/queue/copy/retire 计数、vsync 独立行、late-destination 采样、IOWatchdog/AppleKeyStore 节点检查。
- 结果：
  - backboardd 已连接 display client；
  - 仅 2–3 帧，`vs=0 sub=? arm=0`；
  - `wd=0 aks=0`（服务节点缺失）。

## Stage 16–17 — vsync 与 selector 统计

- 16：vsync 定时器状态（`vst=9 sa=1 ar=3 aa=5`）说明分配/挂接成功，最近一次 arm 时订阅队列为空。
- 17：按 selector 统计（`n/d/f/r`），确认 selector 6 为 3 输入 / 0 输出，返回 Busy 属正常非阻塞路径；selector 15 有真实错误待追踪。

## Stage 18 — wait / completion 配对

- 对原版 wait（slot 176）做精确转发并记录 paired 调用：请求 swap ID、完成 ID 前后值、返回码。
- 结果：
  - 完成 ID 0 → 1 → 2 正常推进；
  - `mode0` 返回 `0xe00002d5`（Busy）符合原版分支；
  - `unmatched=0`，无卡死 wait。
  - **结论：完成握手不是停帧原因。**

## Stage 19 / 19.x — stackshot 用户态栈

- 通过 `stack_snapshot_from_kernel` 采 backboardd / SpringBoard / launchd / lsd 的 kcdata（复制快照，不遍历活动链表）。
- 结果：
  - SpringBoard（p26）11 线程，6 等待；等待线程 `wt=1/2`，内核等待点 `_ipc_mqueue_receive_continue`；
  - 用户栈：libdispatch / libxpc / Foundation（同步 XPC 等回复）；
  - 结论：**不是 GPU/vsync 等待，是同步 XPC/Mach IPC 等某服务回复。**

## Stage 20 — lsd 多实例

- 支持按 page 选择：0 backboardd、1 SpringBoard、2 launchd、3 lsd instance1、4 lsd instance2；同名 lsd 实例分开记录。
- 首测 page3：lsd pid 29，6 线程（3 运行 / 3 等待），选中 worker 阻塞在 Mach IPC；用户栈落在 libsystem_kernel、TelephonyUtilities、FMIPCore。
- `domain=unknown`、`omit=10` 如实输出。

## Stage 21 — 调用者 LR / owner batch

- 记录 owner 进程的调用者 LR 与所属镜像，捕获到：
  - `libsystem_kernel: __posix_spawn`（进程创建路径）
  - `CoreFoundation`
  - `BaseBoard`
- 说明等待链与“服务拉起/查询”相关，仍需端口接收方身份来定位具体服务。

## 仍未完成

- 等待端口的**接收方 pid/comm**（`ipc_port → ip_receiver`）
- launchd job 状态对照（哪个服务没起/崩了）
- GPU/Metal、IOHID 触摸、USB、音频、电池等
