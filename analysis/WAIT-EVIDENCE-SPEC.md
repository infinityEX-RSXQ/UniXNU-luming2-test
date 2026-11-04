# DSH → Codex：Stage19「用户态等待证据」采集规格

日期：2026-11-03。依据 Stage18 r2 结果（completion ID 已推进 0→1→2，mode0 Busy 合法，wait 均返回）——**完成握手不是根因**，下一步必须拿到 backboardd / SpringBoard 的**线程等待点**。

## 0. 目标（只回答一个问题）
> 这两个进程在 15 秒快照时，**有没有线程在等、等什么对象、停在哪个内核/用户态地址**。

不能回答“是否正常”，只能回答“观测到等待/运行状态”。

## 1. 先做可行性检查（不改代码）
在现有 kernel.macho 上确认：
- `nm -m kernel.macho | grep -i kdbg_get_thread_backtrace`（或 `kdbg_thread_backtrace`）
- `grep CONFIG_KDBG` 配置是否 `y`
- `nm | grep -i stackshot_snapshot`
- `nm | grep -i machine_thread_get_state` / `thread_getstatus`

三种情况：
- **A. kdbg 可用** → 采内核栈 PC/LR（首选）
- **B. 只有 stackshot 可用** → 采 stackshot 的 thread 快照（注意权限/缓冲）
- **C. 都不可用** → 只采 `thread->state / wait_event / 最后 syscall`，输出 `stk=none`

**不要为了“有栈”去伪造接口或改原版调度。**

## 2. 采样对象
- 按**进程名**匹配，不硬编码 pid：`SpringBoard`、`backboardd`（Stage18 观察到 p58）
- 同时记录 `launchd`(1) 作为对照
- 每个进程最多 **8 个线程**，优先 `TH_WAIT`/`TH_UNINT`，其次运行中

## 3. 每个进程输出（≤88 字符/行）
```
CDX19 proc name=backboardd pid=58 th=12 run=3 wait=9 stk=ok
CDX19 t tid=123 st=2 w=fffffff00abc1234 sc=42
CDX19 t tid=124 st=1 w=0 sc=...
CDX19 pc k=fffffff007xxxxxx u=1a2b3c4d5e6f lr=...
CDX19 ref iomfb=... vsyncq=... surf=... gpu=...
```
- `st`：0=run, 1=wait, 2=uninterruptible（按 XNU 定义映射）
- `w`：`thread->wait_event` 原值（0 表示未等）
- `sc`：该线程最后一次 syscall 号（`uthread` 字段，需按 21H461 偏移核验）
- `pc/lr`：kdbg 内核栈顶 PC/LR；`u=` 用户态 PC/LR（若可取）
- `ref`：把已知对象地址一并打印，方便比对：IOMFB 实例、vsync 队列、IOSurface/fence、GPU channel

## 4. 采样时机
- 15 秒（与现有 snapshot 同点）采第一次
- 30 秒再采第二次（若进程仍存活）→ 用两次状态区分“瞬时等待”与“长期卡死”
- 每次只输出**汇总 + 最多每进程 4 行线程**

## 5. 与现有诊断的关系
- 保留 CDX11/12/13/14/18 全部行，不删除
- CDX19 放在最后，总长度控制在 900B 内（panic reason 1024B）
- 不改变 selector 返回值、不伪造订阅者、不重写 completed ID

## 6. 关键判定逻辑（写进报告，不要只给原始值）
1. 若 backboardd/SpringBoard 有线程 `st=wait` 且 `w` 等于 **vsync 队列/通知端口**地址 → 指向 vsync/通知链路
2. 若 `w` 等于 **IOSurface/fence/GPU channel** → 指向渲染/GPU
3. 若全部 `st=run` 但 copy 不增加 → 说明不是等待，而是 producer 没提交/被门控
4. 若 `stk=none` → 明确写“无法取得栈”，不能推断进程正常
5. `wait_event` 地址需结合符号化（kernel.macho + slide）判断归属，不能凭低 32 位猜

## 7. 报告格式（给 DSH 的）
- 一行结论：等待点是否被观测到
- 表格：进程 / 线程数 / run/wait/unint / 关键 wait_event 符号
- 原始 CDX19 行
- 明确列出“未取得”的项（`stack=unavailable`、`user_pc=unavailable` 等）

## 8. 边界
- 不要用“没记录到退出”证明进程健康
- 不要在 provider 的 workloop 里直接遍历其他进程线程而破坏锁；用 `proc_lock`/`task_lock` 的正确顺序，或使用 stackshot 现成接口
- 采样必须 best-effort；拿不到就如实 `unavailable`

## 9. 已在本机 Stage18 内核符号表核实（2026-11-03）
对 `builds/stage18-wait-41544084405/kernel.macho`（SHA256 `8ba11aab…`）扫描 symtab：
- `_machine_thread_get_state`：**存在**
- `_thread_getstatus` / `_thread_getstatus_to_user`：**存在**
- `kdbg_get_thread_backtrace` / `kdbg_thread_backtrace`：**不存在**
- `stackshot_snapshot`：**不存在**

因此 Stage19 默认只能走 **C 方案**：
- 采 `thread->state`、`thread->wait_event`、可安全取得的 `uthread` 最后 syscall
- `thread_getstatus()` 取状态（TH_RUN/TH_WAIT/TH_UNINT）
- 用户态 PC/LR 仅在能安全持锁取 `machine_thread_get_state` 时输出；否则标 `u=unavail`
- **不要**为了栈去 patch 原版 kdbg/stackshot；若需要栈，应单独评估启用 CONFIG_KDBG 的构建成本
