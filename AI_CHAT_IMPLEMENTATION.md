# ScoreHub AI 智能问答系统 — 实施总结

## 概述

将本地 Ollama 大模型（`qwen2.5:7b`, `http://localhost:11434`）集成到 ScoreHub Spring Boot 项目中，前端 Vue 3 聊天组件实现打字机流式输出效果。

---

## 架构设计

```
┌─────────────────┐     POST /api/chat/ask     ┌────────────────┐     stream:false      ┌──────────────────┐
│   Vue 3 前端     │ ──────────────────────────> │  Spring Boot   │ ────────────────────> │  Ollama (11434)  │
│  AIChatFloat.vue │ <────────────────────────── │  AIChatController│ <──────────────────── │  qwen2.5:7b      │
│                 │   JSON {success, content}   │                │   完整回答 (单次JSON)  │                  │
│  打字机逐字渲染  │                              │  AIChatServiceImpl│                       │                  │
└─────────────────┘                             └────────────────┘                      └──────────────────┘
```

- **后端**：非流式调用 Ollama（`stream: false`），等待完整回答后一次性返回
- **前端**：收到完整文本后，用 `requestAnimationFrame` 逐字追加到 DOM，模拟流式输出

---

## 项目文件变更

### 新增文件

| 文件 | 用途 |
|------|------|
| `src/main/java/com/scorehub/config/OllamaConfig.java` | OkHttpClient Bean 配置 + 模型参数 |
| `src/main/java/com/scorehub/dto/ChatRequest.java` | 用户消息 DTO：`{ message: string }` |
| `src/main/java/com/scorehub/dto/OllamaChatRequest.java` | Ollama 请求 DTO：model + messages + stream |
| `src/main/java/com/scorehub/dto/OllamaChatResponse.java` | Ollama 响应 DTO：model + message + done + ... |
| `src/main/java/com/scorehub/dto/OllamaMessage.java` | 消息体：`{ role, content }` |
| `src/main/java/com/scorehub/service/AIChatService.java` | AI 聊天服务接口 |
| `src/main/java/com/scorehub/service/impl/AIChatServiceImpl.java` | 服务实现：chat() + chatStream() |
| `src/main/java/com/scorehub/controller/AIChatController.java` | REST 控制器：/ask + /stream |

### 修改文件

| 文件 | 变更内容 |
|------|---------|
| `pom.xml` | 新增 OkHttp 4.12.0、jackson-databind 依赖 |
| `src/main/resources/application.properties` | 新增 `ollama.base-url`、`ollama.model` 配置项 |
| `frontend/src/components/AIChatFloat.vue` | 移除火山引擎 API 调用；新增打字机流式渲染；修复双重头像 bug |
| `frontend/src/views/ScoreHubView.vue` | 引入 AIChatFloat 组件 |

---

## Bug 修复历程（5 轮排查）

### 第 1 轮：中文乱码 + 无流式输出

**现象**：返回 `"浣犲ソ..."` 乱码，前端无流式效果。

**根因**：
- `InputStreamReader(body.byteStream())` 未指定字符集 → Windows 默认 GBK 解码 UTF-8 数据
- JSON 解析失败 → catch 块静默吞掉所有 SSE 事件

**修复**：
- 改用 `InputStream` 直接读取字节 + `new String(buffer, 0, bytesRead, StandardCharsets.UTF_8)` 显式指定 UTF-8

### 第 2 轮：仍然无流式输出（SSE → Tomcat 缓冲区）

**现象**：乱码修复后流式仍不工作，前端只能看到三个点波动后直接输出完整结果。

**根因**：
- `OutputStream.flush()` 只刷新 Java I/O 缓冲区
- **Tomcat/Servlet 容器**有独立的 Response Buffer，默认累积数据直到响应结束才推送

**修复**：
- 添加 `response.setBufferSize(0)`（禁用 Servlet 缓冲）
- 添加 `out.flush()` + `response.flushBuffer()` 双重刷新
- SSE 格式事件实时写入

### 第 3 轮：双重头像

**现象**：AI 回答时出现两个 🤖 头像，回答完成后恢复为一个。

**根因**：
- 模板中 `v-for` 循环渲染空 assistant 消息（含 🤖 头像）
- 另有独立 `v-if="isLoading"` 块也渲染 typing 动画（含另一个 🤖 头像）

**修复**：
- 删除独立的 `v-if="isLoading"` typing 块
- 将 typing 点动画整合进 `v-for` 消息循环，通过 `v-if="msg.content"` / `v-else` 切换

### 第 4 轮：SSE 方案彻底放弃 → Plan C

**现象**：所有 SSE 缓冲优化后，流式仍不工作。

**根因分析**：
- 项目 Spring Boot 2.7 环境可能存在不可见的上游缓冲层（反代、安全过滤器等）
- `nextTick`（微任务级别调度）无法强制浏览器绘制

**修复**：改为"后端全量返回 + 前端打字机模拟"方案。

### 第 5 轮：打字机仍未逐字输出

**现象**：日志显示函数执行完成（`[typewriter] start/done`），但视觉上一次弹出全部文本。

**根因**：
- `targetMsg.content = ...` 修改 Vue ref 内嵌对象属性 → 依赖追踪可能失效
- `await nextTick()` 只是 JS 微任务，浏览器在微任务之间**不会重绘**

**最终修复（双重 rAF）**：
```typescript
// 通过展开运算符创建新对象写回数组，确保 Vue 3 Proxy 响应式更新
messages.value[idx] = {
  ...messages.value[idx],
  content: fullText.slice(0, i + 1),
  timestamp: new Date()
};

// 双重 requestAnimationFrame 强制浏览器帧间绘制
const waitForPaint = () => new Promise<void>(resolve => {
  requestAnimationFrame(() => {
    requestAnimationFrame(() => resolve());
  });
});
```

---

## 当前方案详解

### 后端：POST `/api/chat/ask`

**核心代码**：`AIChatServiceImpl.chat()`

```java
// stream: false → Ollama 一次性返回完整 JSON
OllamaChatRequest request = buildRequest(userMessage);
request.setStream(false);

Request httpRequest = new Request.Builder()
    .url(ollamaBaseUrl + "/api/chat")
    .post(RequestBody.create(requestBody, JSON_MEDIA_TYPE))
    .build();

try (Response ollamaResponse = okHttpClient.newCall(httpRequest).execute()) {
    OllamaChatResponse chatResponse = objectMapper.readValue(body.string(), ...);
    return chatResponse.getMessage().getContent();
}
```

**返回格式**：
```json
{
  "success": true,
  "content": "你好！我是ScoreHub综合评分社交平台的AI智能助手..."
}
```

**System Message**（控制 AI 身份）：
```
你是ScoreHub综合评分社交平台的AI智能助手，请用中文回答用户的问题。
```

### 前端：打字机效果

**核心代码**：`AIChatFloat.vue → typewriterEffect()`

```typescript
// 1. 用展开运算符创建新对象写回数组 → 触发 Vue 3 响应式
messages.value[idx] = {
  ...messages.value[idx],
  content: fullText.slice(0, i + 1),
  timestamp: new Date()
};

// 2. double-rAF 确保浏览器已完成当前帧绘制
await waitForPaint();

// 3. 50ms 停顿让用户看清每个字
await new Promise(r => setTimeout(r, 50));
```

**渲染时间估算**：假设 100 字回答 → 100 × (2帧约33ms + 50ms) ≈ 8.3秒

---

## 利弊分析

### ✅ 优点

| 优点 | 说明 |
|------|------|
| **100% 可靠** | 不依赖 SSE 协议栈任何一层的缓冲行为（Servlet/反代/浏览器），普通 JSON POST 在任何环境都能通过 |
| **无需依赖升级** | Spring Boot 2.7 + Java 8 即可运行，无需升级到 Spring Boot 3.x 的 Spring AI |
| **实现简单** | 后端代码量约 30 行（非流式调用），前端打字机约 20 行 |
| **易于调试** | 后端返回可被 Chrome DevTools Network 面板完整查看，无需观察 EventStream |
| **视觉效果好** | double-rAF 保证每个字符的渲染帧独立，配合 50ms 延迟效果流畅自然 |
| **旧端点保留** | `/api/chat/stream` SSE 端点未删除，未来可随时切回 |

### ❌ 缺点

| 缺点 | 说明 | 影响程度 |
|------|------|---------|
| **首字延迟大** | 必须等 Ollama 生成完**全部**回答后才开始渲染 | 中等（长回答等待明显） |
| **总耗时增加** | 等 Ollama 完整输出 + 前端逐字渲染 → 用户感知的总等待时间更长 | 中等 |
| **无真正的 "生成中" 体验** | 用户在看打字机效果前有长时间空白（只有三个点 loading 动画） | 高（用户体验关键指标） |
| **无终止能力** | 无法中途停止生成（Ollama 已经在后台算完了） | 低 |
| **浪费 Ollama 流式能力** | Ollama 支持 `stream:true` 逐 token 输出，但我们没利用 | 低 |
| **长文本阻塞** | 如果 Ollama 生成 2000+ 字，前端需等待全部完成后才渲染 | 高 |

---

## 未来优化建议

### 方案 A：混合模式（推荐）

1. 后端使用 `stream: true` 调 Ollama，在内存中收集 token 流
2. 每攒够 N 个字符（如 5 个字）作为一个 SSE 事件发给前端
3. 前端收到后追加渲染，不等完整回答

**优势**：兼顾"真实流式首字低延迟"和"SSE 可靠性（批量推送不容易被缓冲）"

### 方案 B：升级技术栈

将 Spring Boot 升级到 3.x + Java 17，使用 **Spring AI** 或 **Spring WebFlux** 的 `Flux<String>` + Reactive SSE，从根源解决 Servlet 缓冲问题。

### 方案 C：WebSocket

改用 WebSocket 全双工通信，天然绕过 HTTP SSE 缓冲链。

---

## 配置参数速查

| 参数 | 位置 | 默认值 | 说明 |
|------|------|--------|------|
| `ollama.base-url` | `application.properties` | `http://localhost:11434` | Ollama 服务地址 |
| `ollama.model` | `application.properties` | `qwen2.5:7b` | 使用的模型名称 |
| `readTimeout` | `OllamaConfig.java` | 120 秒 | OkHttp 读超时 |
| `typing delay` | `AIChatFloat.vue` | 50ms | 打字机每字停顿间隔 |
| `rAF 双重帧` | `AIChatFloat.vue` | ~33ms(2帧) | 强制浏览器绘制等待 |

---

## 启动命令

```bash
# 1. 启动 Ollama（确保 qwen2.5:7b 已拉取）
ollama serve

# 2. 启动后端
cd d:\Workspace\scoreHub
mvn spring-boot:run

# 3. 启动前端
cd d:\Workspace\scoreHub\frontend
npm run dev
```

前端访问 `http://127.0.0.1:5173`，点击右下角 🤖 图标开始对话。
