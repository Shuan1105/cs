# TinyCar WebSocket 25Hz 断连问题与修改说明

---

## 1. 问题现象

> 使用 Python API 以固定频率向小车发送 WebSocket 控制指令 → ESP32（AsyncWebSocket）接收 → 执行电机控制 → 回包 → Python 收到反馈

- 测试中发现：
  - **25 Hz**：经常出现断连
  - **22–25 Hz**：仍会出现断连（频率越高越明显）
  - **≤ 21 Hz**：连续测试 **10–20 分钟**未出现断连（相对稳定）

- 系统在高频时触发： **吞吐瓶颈/调度阻塞**

---

## 2. 根因分析

### 修改前 AsyncWebSocket 的机制
AsyncWebSocket/AsyncTCP 的 `WS_EVT_DATA` 事件回调线程（网络事件线程）里做“重活”：

- 构造/拷贝字符串 `String`
- ArduinoJson 反序列化（`deserializeJson`）
- 立即执行电机控制（`setWheel` / `drive`）
- 高频回包 `client->text(...)`
- 高频 `Serial.print(...)`

回调耗时增加且存在抖动：

1. 新包到达时无法及时被网络栈读取与处理 → **缓冲/队列积压**
2. 积压叠加 WiFi 的突发到包（burst） → **缓冲/队列溢出或超时**
3. 连接被认为“不健康” → **close/reset → 断连**

---

## 3. 结构性优化

核心思想：

> 高频控制类消息通常只关心“最新状态”，修改后仅保留 **最新一条完整消息**，新包覆盖旧包，避免在处理不及时形成队列积压；  
**把“重活”从 WebSocket 回调线程搬到自定义任务线程（update）里**，让网络回调只做简单轻量任务：收包、组包、保存“最新命令”。

### 3.1 修改前
- `WS_EVT_DATA` 回调：**收包 → 解析 JSON → 执行电机控制 → 回包**  
缺点：回调耗时长且抖动大，易造成网络栈积压与断连

### 3.2 修改后
- `WS_EVT_DATA` 回调：**只收包-组帧-保存最新消息（覆盖旧消息）**
- `wsCommTask`（固定周期）调用 `tinyCarComm->update()`：
  1) 取出最新消息  
  2) 解析 JSON  
  3) 更新目标控制量（targetLeft / targetRight 或 targetVx / targetVy / targetW）  
  4) 按固定周期驱动电机 + 超时保护（断线自动刹车）

### 3.3 任务循环使用 `vTaskDelay` + 更高 update 频率
将 `.ino` 的循环从 `delay(50)` 改为：

```cpp
for (;;) {
  if (tinyCarComm != nullptr) {
    tinyCarComm->update();
  }
  vTaskDelay(pdMS_TO_TICKS(20)); // 50Hz
}
```

提升 `update()` 的执行频率有利于更快消费最新命令

### 3.4 修改测试效果
![测试25Hz频率20分钟情况](test.png)
---

## 4. 具体修改点清单

### 4.1 TinyCarComm：回调只存消息，update 解析与控制
- 新增：接收缓冲与“最新消息”结构（`rxLatest`/`rxLatestReady`）
- 新增：分片拼包支持（处理 WebSocket 可能的 fragmented frame）
- 新增：控制目标变量（targetLeft/targetRight 等）与控制模式（WHEEL/MOVE/STOP）
- 新增：控制超时保护（`cmdTimeoutMs`，默认 250ms）
- 调整：`processCommand/handleWheelControl/handleMovement` 在 **update 线程**里执行
- 调整：电机驱动改为 `applyControl()` 固定周期执行，而不是回调即时执行

### 4.2 wheel/move 默认不回 ok（开关）
- 将高频控制回包作为可选项（`wheelAck`/`moveAck`），默认关闭

### 4.3 wsCommTask 频率调整
- 使用 `vTaskDelay(pdMS_TO_TICKS(20))`，将 update 调度稳定在 50Hz

---

## 5. 修改前后对比

| 对比项 | 修改前 | 修改后 | 结果影响 |
|---|---|---|---|
| WS_EVT_DATA 回调任务 | 收包 + JSON解析 + 控制电机 + 回包（开关） | **只收包-组帧-保存最新消息** | 回调“快进快出”，减少网络线程阻塞 |
| 控制策略 | 处理不过来可能形成积压 | **最新覆盖旧（只执行最新目标）** | 抑制延迟堆积，降低拥塞风险 |
| 执行电机控制 | 网络事件回调中 | `update()` 线程内固定周期执行 | 实时控制更可预测、更平滑 |
| update 调度 | delay(50) ≈ 20Hz | vTaskDelay(20ms)=50Hz | \ |
| 高频回包 | 可能每条回 ok | 默认不回（可开关） | 降低反向流量与发送队列压力 |
| 稳定频率上限 | ~21Hz | **25Hz 稳定** | 断连问题解决 |

---


## 6. 结论

断连根本原因：**在 AsyncWebSocket 数据回调中进行重计算/控制导致网络事件线程阻塞，从而在 22–25Hz 临界频率触发 TCP/WebSocket 缓冲积压与断连**。  
“结构性改法”解决断连问题：**回调只存消息 + update 线程解析与控制 + 最新覆盖旧 + 更高频稳定调度，25Hz** 。

---

**更新时间：** 2026-01-17
