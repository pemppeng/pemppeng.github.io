# OpenClaw 队列模式（Queue Mode）详解

**文档定位**：深入解析 OpenClaw 的消息队列处理机制  
**版本**：v1.0  
**更新日期**：2026年2月2日

---

## 一、队列模式概述

OpenClaw 的**队列模式（Queue Mode）**是消息处理的核心机制，决定了当用户连续发送多条消息时，系统如何**排队、合并、丢弃或打断**正在处理的任务。

### 1.1 为什么需要队列模式？

**场景问题**：
- 用户在 3 秒内连续发了 5 条消息
- AI 正在处理第 1 条时，第 2-5 条又到了
- 该如何处理这些"堆积"的消息？

**解决方案**：
- 通过队列模式控制消息处理策略
- 平衡"实时性"与"完整性"
- 避免消息过载导致的混乱

---

## 二、五种队列模式详解

### 2.1 steer 模式（智能合并，默认）

**行为特点**：
- ✅ **智能合并**：将队列中的多条消息合并成一条
- ✅ **打断旧任务**：新消息到达时，打断正在处理的旧任务
- ✅ **实时响应**：始终处理最新的用户意图

**实现逻辑**：
```typescript
// 伪代码示意
function steerMode(messages: Message[]) {
  // 1. 如果正在处理旧任务，打断它
  if (currentTask.isRunning) {
    currentTask.interrupt();
  }
  
  // 2. 合并队列中的所有消息
  const mergedContent = messages.map(m => m.content).join("\n");
  
  // 3. 发送给 AI 处理最新意图
  return ai.process(mergedContent);
}
```

**适用场景**：
- 💬 **实时对话**：用户连续追问，需要立即响应最新问题
- 🔄 **快速迭代**：用户不断修正指令，只需处理最终版本
- ⚡ **高实时性**：客服场景，用户期待秒回

**示例**：
```
用户: "查一下今天的销售额"  ← 开始处理
用户: "不对，查昨天的"      ← 打断旧任务，处理新指令
用户: "算了，查本周的"      ← 再次打断，处理最新指令
AI: "本周销售额是 100 万"   ← 只回复最后一条
```

---

### 2.2 followup 模式（顺序处理）

**行为特点**：
- ✅ **顺序处理**：按消息到达顺序依次处理
- ✅ **不打断**：新消息不中断正在进行的任务
- ✅ **队列排队**：消息进入队列，逐个执行

**实现逻辑**：
```typescript
function followupMode(messages: Message[]) {
  // 1. 将消息加入队列
  queue.push(...messages);
  
  // 2. 按顺序处理，不跳过
  for (const msg of queue) {
    await ai.process(msg);  // 等待完成再处理下一条
  }
}
```

**适用场景**：
- 📋 **批量任务**：用户提交多个独立任务，需要全部完成
- 📝 **分步指导**：教学场景，逐步执行每个步骤
- 📊 **报表生成**：生成多张报表，每张都需要时间

**示例**：
```
用户: "生成日报"     ← 加入队列，开始处理
用户: "生成周报"     ← 加入队列，等待中
用户: "生成月报"     ← 加入队列，等待中
AI: "日报完成..."    ← 第 1 个完成
AI: "周报完成..."    ← 第 2 个完成  
AI: "月报完成..."    ← 第 3 个完成
```

---

### 2.3 collect 模式（收集批量响应）

**行为特点**：
- ✅ **收集模式**：累积一定数量的消息后再统一处理
- ✅ **批量响应**：一次性回复多条消息的结果
- ✅ **防刷屏**：避免 AI 频繁回复，合并成一条

**实现逻辑**：
```typescript
function collectMode(messages: Message[]) {
  // 1. 累积消息到缓冲区
  buffer.push(...messages);
  
  // 2. 等待达到阈值或超时
  if (buffer.length >= threshold || timeout()) {
    // 3. 批量处理所有消息
    const batchResult = await ai.processBatch(buffer);
    
    // 4. 合并成一条回复
    return mergeResponses(batchResult);
  }
}
```

**适用场景**：
- 🗳️ **投票收集**：收集群成员的投票，统一统计
- 📝 **意见征集**：收集团队反馈，批量整理
- 📋 **清单勾选**：逐项确认，最后统一回复

**示例**：
```
用户1: "我选方案 A"   ← 收集
用户2: "我选方案 B"   ← 收集
用户3: "我选方案 A"   ← 收集
用户4: "我选方案 A"   ← 达到阈值，触发处理
AI: "投票结果：
    - 方案 A: 3 票
    - 方案 B: 1 票
    - 最终选择：方案 A"
```

---

### 2.4 queue 模式（严格队列）

**行为特点**：
- ✅ **严格队列**：FIFO（先进先出）顺序执行
- ✅ **容量限制**：队列有上限（cap），超过则丢弃
- ✅ **丢弃策略**：可配置丢弃旧消息或新消息

**实现逻辑**：
```typescript
function queueMode(messages: Message[]) {
  // 1. 检查队列容量
  if (queue.length + messages.length > cap) {
    // 2. 应用丢弃策略
    switch (dropPolicy) {
      case "old":
        queue.shift();  // 丢弃最旧的消息
        break;
      case "new":
        return;  // 丢弃新消息
      case "summarize":
        summarizeDroppedMessages();  // 摘要丢弃的消息
        break;
    }
  }
  
  // 3. 加入队列，顺序执行
  queue.push(...messages);
  processQueueSequentially();
}
```

**配置参数**：
- **cap**：队列容量上限（如 5 条）
- **dropPolicy**：丢弃策略
  - `old`：丢弃最旧的消息
  - `new`：丢弃新消息（保留旧任务）
  - `summarize`：对丢弃的消息生成摘要

**适用场景**：
- ⚠️ **重要任务**：不能遗漏任何消息，必须按顺序处理
- 🎫 **工单系统**：每个请求都必须被处理
- 📞 **客服队列**：客户排队等待，不能插队

**示例**（cap=3, drop=old）：
```
队列状态: [msg1, msg2, msg3]  ← 已满
用户发送: msg4
AI: 丢弃 msg1（最旧）
队列状态: [msg2, msg3, msg4]
AI: "正在处理 msg2..."
```

---

### 2.5 interrupt 模式（紧急打断）

**行为特点**：
- ✅ **立即打断**：新消息到达时，立即停止当前任务
- ✅ **丢弃旧任务**：未完成的旧任务直接丢弃
- ✅ **最高优先级**：始终处理最新消息

**实现逻辑**：
```typescript
function interruptMode(newMessage: Message) {
  // 1. 立即停止当前任务
  if (currentTask.isRunning) {
    currentTask.abort();  // 强制中止
    droppedTasks.push(currentTask);  // 记录被丢弃的任务
  }
  
  // 2. 立即处理新消息
  return ai.process(newMessage);
}
```

**适用场景**：
- 🚨 **紧急场景**：用户发送"停止！"或"取消"
- 🛑 **错误修正**：用户发现指令错误，立即纠正
- ⚡ **高优先级**：重要通知，必须立即响应

**示例**：
```
用户: "删除所有文件"     ← 开始执行（危险操作！）
用户: "停止！别删除！"   ← 立即打断，停止删除
AI: "已停止操作，未删除任何文件"
```

---

## 三、高级模式组合

### 3.1 steer+backlog / steer-backlog 模式

**特点**：steer 模式的变体，处理历史积压消息

```typescript
// steer+backlog: 处理最新消息 + 积压的历史消息
function steerBacklogMode(messages: Message[]) {
  const latest = messages.pop();  // 最新消息
  const backlog = messages;        // 历史积压
  
  // 合并处理：先处理积压，再处理最新
  return ai.process({
    backlog: backlog,
    current: latest
  });
}
```

**适用场景**：
- 用户一段时间没看消息，积累了多条
- 需要了解上下文后再回复最新问题

---

## 四、队列配置参数

### 4.1 完整配置示例

```typescript
// config.json
{
  "messages": {
    "queue": {
      "mode": "steer",           // 队列模式
      "cap": 5,                  // 队列容量上限
      "debounceMs": 1000,        // 防抖时间（毫秒）
      "dropPolicy": "summarize"  // 丢弃策略
    }
  }
}
```

### 4.2 参数说明

| 参数 | 类型 | 说明 | 默认值 |
|------|------|------|--------|
| **mode** | string | 队列模式 | "steer" |
| **cap** | number | 队列容量上限 | 5 |
| **debounceMs** | number | 防抖等待时间（ms） | 1000 |
| **dropPolicy** | string | 丢弃策略 | "old" |

### 4.3 按渠道独立配置

```typescript
// 不同渠道使用不同队列策略
{
  "messages": {
    "queue": {
      "byProvider": {
        "whatsapp": "steer",      // WhatsApp 实时对话
        "telegram": "followup",   // Telegram 批量任务
        "discord": "collect",     // Discord 投票收集
        "email": "queue"          // 邮件严格队列
      }
    }
  }
}
```

---

## 五、队列模式对比表

| 模式 | 处理顺序 | 打断旧任务 | 合并消息 | 容量限制 | 适用场景 |
|------|---------|-----------|---------|---------|---------|
| **steer** | 最新优先 | ✅ 打断 | ✅ 合并 | ❌ 无 | 实时对话 |
| **followup** | 先进先出 | ❌ 不打断 | ❌ 不合并 | ❌ 无 | 批量任务 |
| **collect** | 批量处理 | ❌ 不打断 | ✅ 合并 | ✅ 有 | 信息收集 |
| **queue** | 严格 FIFO | ❌ 不打断 | ❌ 不合并 | ✅ 有 | 重要任务 |
| **interrupt** | 最新优先 | ✅ 强制打断 | ❌ 不合并 | ❌ 无 | 紧急场景 |

---

## 六、实际应用建议

### 6.1 按场景选择模式

| 场景 | 推荐模式 | 理由 |
|------|---------|------|
| 客服对话 | `steer` | 实时响应，处理最新问题 |
| 报表生成 | `followup` | 批量生成，不遗漏任何报表 |
| 群投票 | `collect` | 收集所有投票，统一统计 |
| 工单处理 | `queue` | 严格顺序，防止插队 |
| 紧急控制 | `interrupt` | 立即停止危险操作 |

### 6.2 动态切换模式

用户可以在对话中通过指令动态切换队列模式：

```
用户: "/queue followup"     ← 切换到顺序处理模式
用户: "生成日报"            ← 加入队列
用户: "生成周报"            ← 加入队列
用户: "生成月报"            ← 加入队列
AI: 依次处理三个报表...
```

---

## 七、技术实现要点

### 7.1 防抖机制（Debounce）

**作用**：防止用户连续输入时频繁触发处理

```typescript
// 实现逻辑
let debounceTimer;
function onMessageReceived(msg) {
  clearTimeout(debounceTimer);
  debounceTimer = setTimeout(() => {
    processQueue();  // 防抖后才处理
  }, debounceMs);
}
```

### 7.2 任务中断机制

**作用**：安全地停止正在进行的 AI 任务

```typescript
class Task {
  private abortController = new AbortController();
  
  interrupt() {
    this.abortController.abort();  // 发送中止信号
    this.cleanup();                 // 清理资源
  }
  
  async run() {
    try {
      await ai.generate({
        signal: this.abortController.signal  // 可中断
      });
    } catch (e) {
      if (e.name === 'AbortError') {
        console.log('任务被中断');
      }
    }
  }
}
```

### 7.3 消息去重

**作用**：避免重复处理相同内容的消息

```typescript
function shouldSkipDuplicate(newMsg: Message, queue: Message[]) {
  return queue.some(msg => 
    msg.content === newMsg.content && 
    msg.author === newMsg.author
  );
}
```

---

## 八、总结

OpenClaw 的队列模式是**消息调度的核心机制**，通过 5 种模式平衡了：

1. **实时性 vs 完整性**：steer/interrupt 优先实时，followup/queue 保证完整
2. **响应速度 vs 任务准确**：快速打断可能损失上下文，顺序处理保证质量
3. **用户体验 vs 系统负载**：collect 合并减少 API 调用，降低 costs

**最佳实践**：
- 默认使用 `steer` 模式（实时对话）
- 批量任务时切换到 `followup` 模式
- 紧急场景使用 `interrupt` 模式
- 按渠道配置不同策略，优化体验

---

**文档结束**

*本文档详细解析了 OpenClaw 的队列处理机制，帮助理解消息调度原理*
