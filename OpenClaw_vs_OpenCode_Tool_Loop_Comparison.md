# OpenClaw 与 OpenCode 工具调用循环机制深度对比

**分析日期**：2026年2月2日  
**分析对象**：
- OpenClaw: https://github.com/openclaw/openclaw
- OpenCode: https://github.com/anomalyco/opencode

---

## 一、核心结论（TL;DR）

| 维度 | OpenClaw | OpenCode |
|------|----------|----------|
| **循环类型** | ✅ 是，ReAct 循环 | ✅ 是，ReAct 循环 |
| **循环位置** | 底层库封装 (`pi-agent-core`) | 显式在业务代码中 |
| **循环控制** | 由底层 SDK 管理 | 由 `processor.ts` 显式控制 |
| **可见性** | 对开发者透明 | 代码中清晰可见 `while (true)` |

**两者都是循环，但实现方式不同**：
- **OpenCode**：显式循环，在 `processor.ts` 中直接写 `while (true)`
- **OpenClaw**：隐式循环，由底层 `@mariozechner/pi-agent-core` 库封装

---

## 二、OpenCode 工具调用循环详解

### 2.1 代码位置

**文件**：`packages/opencode/src/session/processor.ts`

### 2.2 循环结构

```typescript
export function create(input: {
  assistantMessage: MessageV2.Assistant
  sessionID: string
  model: Provider.Model
  abort: AbortSignal
}) {
  const toolcalls: Record<string, MessageV2.ToolPart> = {}
  let attempt = 0
  let blocked = false
  
  const result = {
    async process(streamInput: LLM.StreamInput) {
      // ==================== 外层循环 ====================
      while (true) {  // ← 显式的无限循环
        try {
          // 1. 调用 LLM，获取流式输出
          const stream = await LLM.stream(streamInput)
          
          // 2. 处理流式输出中的事件
          for await (const value of stream.fullStream) {
            switch (value.type) {
              // 2.1 工具开始调用
              case "tool-input-start":
                // 创建工具调用记录
                break
                
              // 2.2 实际执行工具
              case "tool-call": {
                // 执行工具，等待结果
                const part = await Session.updatePart({...})
                toolcalls[value.toolCallId] = part
                
                // 检查是否陷入"末日循环"（doom loop）
                // 即反复调用同一个工具、相同参数
                if (lastThree.every(p => 
                  p.type === "tool" && 
                  p.tool === value.toolName &&
                  JSON.stringify(p.state.input) === JSON.stringify(value.input)
                )) {
                  // 请求用户确认
                  await PermissionNext.ask({...})
                }
                break
              }
              
              // 2.3 工具执行完成
              case "tool-result": {
                // 保存工具执行结果
                await Session.updatePart({
                  state: {
                    status: "completed",
                    output: value.output.output
                  }
                })
                delete toolcalls[value.toolCallId]
                break
              }
              
              // 2.4 工具执行出错
              case "tool-error": {
                await Session.updatePart({
                  state: {
                    status: "error",
                    error: value.error
                  }
                })
                if (value.error instanceof PermissionNext.RejectedError) {
                  blocked = shouldBreak  // 用户拒绝，标记阻塞
                }
                delete toolcalls[value.toolCallId]
                break
              }
              
              // 2.5 LLM 输出文本
              case "text-delta":
                // 累加文本输出
                break
                
              // 2.6 步骤完成
              case "finish-step":
                // 一轮思考-行动完成
                break
            }
          }
          
          // 3. 检查循环退出条件
          if (needsCompaction) return "compact"      // 需要压缩上下文
          if (blocked) return "stop"                 // 被权限阻塞
          if (input.assistantMessage.error) return "stop"  // 出错
          return "continue"  // 继续下一轮循环
          
        } catch (e) {
          // 错误处理 + 重试机制
          const retry = SessionRetry.retryable(error)
          if (retry !== undefined) {
            attempt++
            const delay = SessionRetry.delay(attempt)
            await SessionRetry.sleep(delay)
            continue  // 重试，继续循环
          }
          // 不可重试错误，退出循环
          return "stop"
        }
      }  // end while (true)
    }
  }
}
```

### 2.3 循环流程图

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenCode ReAct Loop                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. 用户输入消息                                             │
│     await LLM.stream({ messages, tools })                   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. LLM 思考（Reasoning）                                    │
│     case "reasoning-start/delta/end"                        │
│     显示 AI 的思考过程                                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. LLM 决定调用工具（Acting）                                │
│     case "tool-call":                                       │
│     - 解析工具名称和参数                                      │
│     - 执行工具（如：read_file, search_files）                 │
│     - 等待工具执行完成                                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. 工具执行结果返回给 LLM                                    │
│     case "tool-result":                                     │
│     - 保存执行结果                                           │
│     - 继续流式输出，LLM 观察结果                              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. LLM 继续思考或输出最终答案                                 │
│     case "text-delta":                                      │
│     - 累加文本输出                                           │
│     - 如果是最终答案，case "finish-step"                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
┌─────────────────────────┐   ┌─────────────────────────┐
│  6a. 需要更多工具调用？   │   │  6b. 任务完成？         │
│     回到步骤 3           │   │     退出循环            │
└─────────────────────────┘   └─────────────────────────┘
```

### 2.4 循环特点

**显式控制**：
- ✅ `while (true)` 直接写在业务代码中
- ✅ 循环退出条件清晰可见
- ✅ 重试逻辑（retry）在循环内处理

**ReAct 模式**（Reasoning + Acting）：
- ✅ LLM 先思考（reasoning）
- ✅ 然后决定调用工具（tool-call）
- ✅ 观察工具结果（tool-result）
- ✅ 循环直到任务完成

**防循环机制**：
- ✅ **Doom Loop 检测**：检查是否反复调用同一工具、相同参数
- ✅ **权限拦截**：用户拒绝时设置 `blocked = true`
- ✅ **重试上限**：`attempt` 计数器，避免无限重试

---

## 三、OpenClaw 工具调用循环详解

### 3.1 代码位置与架构

**文件**：
- `src/agents/pi-embedded-runner/run/attempt.ts`（调用入口）
- 实际循环在底层库 `@mariozechner/pi-agent-core` 中

### 3.2 调用链分析

```typescript
// attempt.ts
export async function runEmbeddedAttempt(params) {
  // 1. 创建 Agent Session
  const { session } = await createAgentSession({
    tools: builtInTools,
    customTools: allCustomTools,
    // ...
  })
  
  // 2. 订阅会话事件
  const subscription = subscribeEmbeddedPiSession({
    session: activeSession,
    onToolResult: params.onToolResult,
    // ...
  })
  
  // 3. 发送 prompt（触发循环）
  await activeSession.prompt(effectivePrompt)
  
  // 4. 等待完成（循环由底层库管理）
  await waitForCompactionRetry()
}
```

### 3.3 底层库的工作方式

OpenClaw 使用了 `@mariozechner/pi-agent-core` 库，该库封装了 ReAct 循环：

```typescript
// 伪代码：底层库的工作方式
class AgentSession {
  async prompt(userInput: string) {
    // 内部循环（对调用者透明）
    while (this.shouldContinue()) {
      // 1. 调用 LLM
      const response = await this.llm.generate({
        messages: this.messages,
        tools: this.tools
      })
      
      // 2. 检查是否有工具调用
      if (response.toolCalls) {
        for (const toolCall of response.toolCalls) {
          // 3. 执行工具
          const result = await this.executeTool(toolCall)
          
          // 4. 将结果加入消息历史
          this.messages.push({
            role: "tool",
            content: result,
            toolCallId: toolCall.id
          })
        }
        // 5. 继续循环，让 LLM 观察结果
        continue
      }
      
      // 6. 没有工具调用，输出最终答案
      return response.text
    }
  }
}
```

### 3.4 OpenClaw 的事件订阅机制

虽然循环在底层，但 OpenClaw 通过**订阅模式**暴露工具调用事件：

```typescript
// subscribeEmbeddedPiSession 函数
export function subscribeEmbeddedPiSession(params) {
  // 订阅底层库的事件
  session.on('tool_call', (toolCall) => {
    // 触发前端显示
    params.onToolResult?.(toolCall)
  })
  
  session.on('text_delta', (text) => {
    // 流式显示文本
    params.onPartialReply?.(text)
  })
  
  session.on('finish', () => {
    // 循环结束
  })
}
```

### 3.5 循环流程图（逻辑层面）

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenClaw ReAct Loop                     │
│              （由 pi-agent-core 库封装管理）                   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  1. 用户调用 activeSession.prompt(message)                   │
│     ↓ 进入底层库                                              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  2. 底层库循环开始（不可见）                                    │
│     while (shouldContinue) {                                │
│       const response = llm.generate(messages, tools)        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  3. LLM 返回工具调用请求？                                     │
│     if (response.hasToolCalls) {                            │
│       // 执行工具                                            │
│       const result = executeTool(response.toolCalls[0])     │
│       messages.push({ role: "tool", content: result })      │
│       continue  // 继续循环                                   │
│     }                                                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  4. LLM 返回最终答案                                          │
│     messages.push({ role: "assistant", content: response }) │
│     break  // 退出循环                                        │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  5. 返回结果给用户                                           │
│     await activeSession.prompt() 返回                        │
└─────────────────────────────────────────────────────────────┘
```

### 3.6 循环特点

**封装隐藏**：
- ✅ 循环在底层库 `@mariozechner/pi-agent-core` 中
- ✅ 业务代码只调用 `activeSession.prompt()`
- ✅ 通过事件订阅获取中间状态

**事件驱动**：
- ✅ `subscribeEmbeddedPiSession` 订阅工具调用事件
- ✅ 实时流式显示（`onPartialReply`, `onToolResult`）
- ✅ 支持中止操作（`abortSignal`）

**上下文管理**：
- ✅ `SessionManager` 维护消息历史
- ✅ `guardSessionManager` 包装保护
- ✅ 自动压缩（`compaction`）防止超出 Token 限制

---

## 四、两者循环的详细对比

### 4.1 代码可见性对比

| 特性 | OpenCode | OpenClaw |
|------|----------|----------|
| **循环可见性** | ✅ 显式 `while (true)` | ⚠️ 隐式，在底层库中 |
| **代码位置** | `processor.ts` 业务层 | `pi-agent-core` 库中 |
| **控制粒度** | 精细，可自定义每一步 | 粗粒度，由库管理 |
| **学习成本** | 低，直接阅读源码 | 高，需要理解库的封装 |

### 4.2 循环控制流对比

**OpenCode 显式控制**：
```typescript
// 业务代码直接控制循环
while (true) {
  const stream = await LLM.stream(input)
  
  for await (const event of stream) {
    // 处理每个事件
    if (event.type === "tool-call") {
      await executeTool(event)
    }
  }
  
  // 显式检查退出条件
  if (shouldStop) break
}
```

**OpenClaw 隐式控制**：
```typescript
// 业务代码只触发调用
await session.prompt(input)  // 循环在内部执行

// 通过事件监听获取进度
subscribeEmbeddedPiSession({
  onToolResult: (result) => { /* 处理工具结果 */ }
})
```

### 4.3 ReAct 循环步骤对比

| 步骤 | OpenCode | OpenClaw |
|------|----------|----------|
| **1. 思考（Reasoning）** | ✅ `case "reasoning-*"` | ✅ 底层库处理 |
| **2. 工具调用（Act）** | ✅ `case "tool-call"` | ✅ 底层库处理 |
| **3. 观察（Observe）** | ✅ `case "tool-result"` | ✅ 自动加入 messages |
| **4. 继续/结束** | ✅ 显式 `continue/break` | ✅ 底层库判断 |

### 4.4 工具执行流程对比

**OpenCode 工具执行**：
```
用户输入
    ↓
LLM.stream() → 流式输出
    ↓
检测到 tool-call 事件
    ↓
立即执行工具（await executeTool）
    ↓
等待工具完成 → tool-result 事件
    ↓
继续流式输出（LLM 观察结果）
    ↓
循环或结束
```

**OpenClaw 工具执行**：
```
用户输入
    ↓
session.prompt() → 触发底层循环
    ↓
底层库调用 LLM
    ↓
检测到 tool_calls
    ↓
底层库执行工具
    ↓
将结果加入 messages
    ↓
再次调用 LLM（循环）
    ↓
返回最终结果
```

### 4.5 优缺点对比

**OpenCode 显式循环**：

优点：
- ✅ 代码清晰，易于理解和调试
- ✅ 完全控制每一步流程
- ✅ 可自定义循环逻辑（如 doom loop 检测）

缺点：
- ❌ 代码量较大
- ❌ 需要处理所有边界情况
- ❌ 维护成本较高

**OpenClaw 隐式循环**：

优点：
- ✅ 业务代码简洁
- ✅ 由专业库处理复杂逻辑
- ✅ 易于集成和扩展

缺点：
- ❌ 黑盒，难以理解内部机制
- ❌ 调试困难
- ❌ 自定义能力受限

---

## 五、总结：循环的本质

### 5.1 两者都是 ReAct 循环

**ReAct = Reasoning（思考） + Acting（行动）**

```
┌─────────────────────────────────────────────────────────────┐
│                      ReAct Loop                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. Reasoning（思考）                                       │
│      LLM 分析问题，决定下一步行动                             │
│                      ↓                                      │
│   2. Acting（行动）                                          │
│      调用工具（如：read_file, search_files）                 │
│                      ↓                                      │
│   3. Observing（观察）                                       │
│      获取工具执行结果                                        │
│                      ↓                                      │
│   4. Loop（循环）                                            │
│      回到步骤 1，继续思考，直到任务完成                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 核心差异总结

| 维度 | OpenCode | OpenClaw |
|------|----------|----------|
| **是否有循环** | ✅ 是 | ✅ 是 |
| **循环实现** | 显式 `while (true)` | 隐式，底层库封装 |
| **控制方式** | 业务代码直接控制 | 通过事件订阅间接控制 |
| **灵活性** | 高，可完全自定义 | 中，受限于库的 API |
| **复杂度** | 高，代码量大 | 低，代码简洁 |

### 5.3 设计哲学差异

**OpenCode**：
> "显式优于隐式，控制优于便利"

- 愿意牺牲简洁性换取完全控制
- 适合需要深度定制的场景

**OpenClaw**：
> "封装复杂性，提供简洁 API"

- 牺牲一定灵活性换取开发效率
- 适合快速集成和标准化场景

---

## 六、对开发者的启示

### 6.1 如果你要实现类似功能

**选择显式循环（OpenCode 方式）**：
- 需要完全控制 AI 行为
- 需要自定义重试、错误处理逻辑
- 需要深度调试和优化

**选择隐式循环（OpenClaw 方式）**：
- 追求开发效率
- 使用成熟的 Agent 框架
- 标准化场景，无需特殊定制

### 6.2 关键设计决策

1. **循环退出条件**：
   - 必须有明确的退出条件（避免死循环）
   - 建议：最大轮数限制 + 任务完成检测

2. **工具调用安全**：
   - 防止 doom loop（反复调用同一工具）
   - 用户权限确认（敏感操作）

3. **错误处理**：
   - 工具执行失败的重试策略
   - 网络错误、API 限流的处理

4. **上下文管理**：
   - 防止 messages 无限增长
   - 定期压缩或清理历史

---

**文档结束**

*本文档基于 OpenClaw 和 OpenCode 的实际源代码分析*
