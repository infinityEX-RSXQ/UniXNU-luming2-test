# 下一阶段（Stage 22+）计划与轮次估计

## 已排除的方向

- ❌ CADisplay 为 nil（Stage11 已越过）
- ❌ FBSDisplayMode 参数非法（Stage12 已修复）
- ❌ 2× UI scale 未生效（Stage13 已回读成功）
- ❌ VT 覆盖画面（Stage14.1 已证明 `blank=0`、真实帧复制）
- ❌ completion ID / selector 6 握手（Stage18 已证明正常推进，Busy 合法）

## 当前唯一主线

**SpringBoard 与 lsd 都阻塞在 Mach IPC 收消息**，等待某个服务回复。下一步必须拿到：

1. 等待端口的**接收方 pid / comm**（`ipc_mqueue → ipc_port → ip_receiver`）
2. 同一时刻 launchd 的 job 状态（哪个服务未启动/已崩溃）
3. 四个页（backboardd / SpringBoard / launchd / lsd）一次采全，避免一轮只看一个

## 轮次估计（软件路径可行时）

| 轮次 | 目的 | 可能结果 |
|---|---|---|
| 1 | 端口接收方归属（Stage22） | 确定等谁：backboardd / launchd / 某服务 |
| 2 | 若等的是 launchd job → 查 job 状态与依赖 | 定位未启动/崩溃的服务 |
| 3–5 | 修服务缺口（KeyStore/Watchdog/LaunchServices）后出现下一个等待 | 每修一个暴露下一个 |
| 6–8 | 桌面/锁屏前的收尾 | 不确定 |

- 若最终确认需要 **GPU/Metal**，则不是轮次问题，而是 Adreno/IOGPU 移植（周级）。
- 每轮 ≈ 构建 30–60 分钟 + 刷机验证 10 分钟。

## 原则

- 一次采集尽量覆盖全部候选页，减少轮次
- 未取得的字段输出 `unavailable`，不猜测
- 不伪造服务、不跳过原版断言、不改 completion ID
- 每个结论必须有真机照片/日志证据
