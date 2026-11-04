# 证据索引（摘要）

原始照片、完整日志、Apple 二进制与反汇编**不发布**在本仓库；这里只保留可复核的结论、指标与采集方法。

## 显示链路

| 指标 | 观测值 | 说明 |
|---|---|---|
| validate / copy / retire | 3 / 3 / 3 | 整帧 1236×2750×4 = 13,596,000 B |
| blank | 0 | 不是清黑 |
| 源/目标采样 | 一致（`mismatch=0`, `mutate=0`） | 复制瞬间一致 |
| late vs last | `eq=1`（采样点） | 清屏前目标未被改写 |
| VT 状态 | `own=0 enabled=1 drawn=0`（恢复后） | 恢复前 `before=1/0/1` |

## 显示模式 / 缩放

| 指标 | 观测值 |
|---|---|
| FBSDisplayMode 参数 | `w=1236 h=2750 pref=0 rot=0` |
| refresh | `rateBits=404e000000000000`（60.0 Hz） |
| scale 发布 | `pub=1 prop=1`；`CDX13 ui=2 read=2 rc=0` |
| UI scale | 618×1375 points（2×） |

## 提交者与 IPC

| 指标 | 观测值 |
|---|---|
| display client | backboardd（pid 58）等 |
| 帧数 | 2–3 帧后停止 |
| submit/validate/queue/copy/retire | `ipc=35 vel=3 q=3 cp=3 ret=3`（Stage15） |
| vsync | `vs=0 last=0 sub=? arm=0`；定时器状态 `vst=9 sa=1 ar=3 aa=5 hz=60` |
| selector 6 | 3 输入 / 0 输出；返回 `0xe00002d5`（Busy，合法） |
| selector 15 | 真实错误 `0x2bc`，待追踪 |

## wait / completion

| 指标 | 观测值 |
|---|---|
| 完成 ID | 0 → 1 → 2（正常推进） |
| waits / done | `waits=3 done=2`，`unmatched=0` |
| mode0 Busy | `0xe00002d5`（原版非阻塞分支） |
| 结论 | 完成握手不是停帧原因 |

## 用户态等待

| 进程 | 观测 |
|---|---|
| SpringBoard | 11 线程，6 等待；等待点 `_ipc_mqueue_receive_continue`；用户栈 libdispatch / libxpc / Foundation |
| lsd（launchservicesd） | pid 29，6 线程，3 等待；用户栈 libsystem_kernel / TelephonyUtilities / FMIPCore |
| 等待类型 | `wt=1/2` / `wt=1/18`（Mach IPC） |
| 结论 | 阻塞在**同步 XPC / Mach IPC 等回复**，不是 GPU/vsync |

## 平台服务

| 服务 | 观测 |
|---|---|
| IOWatchdog | IORegistry 无节点；watchdogd 因查找失败退出 |
| AppleKeyStore | IORegistry 无节点；liveactivitiesd 因 Keybag 事件注册返回空退出 |
| 因果关系 | 与桌面停帧**尚未证实**，需端口接收方证据 |

## 采集方法

- panic 摘要短行（≤88 字符/行），panic reason 缓冲 1024B
- kcdata 复制快照（`stack_snapshot_from_kernel`），不遍历活动线程链表
- 所有未知字段输出 `unavailable` / `unknown`，不做推断
