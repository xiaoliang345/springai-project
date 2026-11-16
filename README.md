# springai-project

基于 Spring Boot 4 + Spring AI 2.0 接入 DeepSeek 的对话示例项目，包含 ChatClient 调用、系统提示词设定与多轮会话记忆。

## 技术栈

| 组件 | 版本 |
| --- | --- |
| JDK | 21 |
| Spring Boot | 4.1.2 |
| Spring AI | 2.0.0 |
| DeepSeek Starter | `spring-ai-starter-model-deepseek` |
| Lombok | 由 Spring Boot 管理 |
| Hutool | 5.8.46 |

## 功能说明

- `ChatClient` 统一封装模型调用，通过 `DeepSeekChatModel` 自动装配注入。
- 默认系统提示词：恋爱心理领域专家角色（见 `ChatController.SYSTEM_PROMPT`）。
- 多轮会话记忆：`MessageChatMemoryAdvisor` + Spring AI 自动配置的 `ChatMemory`（`MessageWindowChatMemory`，默认窗口 20 条，超出后按整轮淘汰，SystemMessage 始终保留），按 `chatId` 隔离会话。

## 环境要求

- JDK 21+
- Maven 3.9+（或直接使用项目自带的 `mvnw` / `mvnw.cmd`）
- 可用的 DeepSeek API Key

## 快速开始

### 1. 配置 API Key

`application.yaml` 中通过 `${DEEPSEEK_API_KEY}` 引用密钥，本地开发请把真实 Key 写入 `src/main/resources/application-local.yaml`（该文件已在 `.gitignore` 中，不会提交）：

```yaml
spring:
  ai:
    deepseek:
      api-key: sk-xxxxxxxxxxxxxxxx
      chat:
        model: deepseek-chat
        temperature: 0.3
        max-tokens: 2048
```

也可以通过环境变量或 IDE 运行配置注入：

```
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxx
```

### 2. 启动项目

默认激活 `local` profile，端口 `8080`：

```bash
# Windows
mvnw.cmd spring-boot:run

# macOS / Linux
./mvnw spring-boot:run
```

### 3. 调用接口

```bash
curl "http://localhost:8080/ai/generate?message=我最近和女朋友总因为生活习惯吵架&chatId=1"
```

同一个 `chatId` 的多次请求会带上历史上下文；换一个 `chatId` 即为全新会话。

## 接口说明

| 方法 | 路径 | 参数 | 说明 |
| --- | --- | --- | --- |
| GET | `/ai/generate` | `message`：用户消息<br>`chatId`：会话 ID | 返回模型文本回复 |

参数补充：

- `chatId` 会以字符串形式作为 `ChatMemory.CONVERSATION_ID` 传入 Advisor，是记忆分区键，必须传。
- 记忆存储在 `InMemoryChatMemoryRepository`（进程内），**重启后丢失，多实例不共享**。需要持久化可替换为 JDBC / Redis 仓库。

## 配置说明

| 配置项 | 说明 |
| --- | --- |
| `server.port` | 服务端口，默认 8080 |
| `spring.profiles.active` | 激活的配置环境，当前为 `local` |
| `spring.ai.deepseek.api-key` | DeepSeek API Key |
| `spring.ai.deepseek.chat.model` | 模型名称，需与所接入平台提供的模型名一致 |
| `spring.ai.deepseek.chat.temperature` | 采样温度，0~1，越低越稳定 |
| `spring.ai.deepseek.chat.max-tokens` | 单次回复最大 token 数 |
| `spring.ai.deepseek.chat.thinking.type` | 思考模式开关，`disabled` 时不产生推理过程 |
| `spring.ai.deepseek.chat.reasoning-effort` | 思考深度，仅在思考模式开启时生效 |

> 说明：Spring AI 2.0 中模型参数位于 `spring.ai.deepseek.chat.options.*` 下；若使用自定义模型选项，请确认参数绑定层级，避免配置未生效。

## 日志

日志配置见 `src/main/resources/logback-spring.xml`：

- 控制台输出带级别着色，时间精确到分钟（`yyyy-MM-dd HH:mm`）。
- 文件输出至 `logs/springai-project.log`，按天 + 50MB 滚动，保留 30 天。
- 业务包 `com.oxn.springaiproject` 为 `debug` 级别，其余为 `info`。

使用 Lombok `@Slf4j` 输出日志：

```java
log.info("调用模型, chatId={}, message={}", chatId, message);
```

## 项目结构

```
springai-project
├── src/main/java/com/oxn/springaiproject
│   ├── SpringaiProjectApplication.java   # 启动类
│   └── ChatController.java               # 对话接口 + ChatClient 装配
├── src/main/resources
│   ├── application.yaml                  # 主配置
│   ├── application-local.yaml            # 本地配置（已忽略，含 API Key）
│   └── logback-spring.xml                # 日志配置
├── .gitattributes                        # 换行符规范
├── .gitignore
├── mvnw / mvnw.cmd                       # Maven Wrapper
└── pom.xml
```

## 注意事项

- `application-local.yaml` 已加入 `.gitignore`，请勿提交真实密钥。
- `deepseek-flash` 等模型名需与所用平台（官方 API 或第三方网关）文档一致；DeepSeek 官方模型为 `deepseek-chat` 与 `deepseek-reasoner`。
- `MessageChatMemoryAdvisor` 必须通过 Advisor 挂载到 `ChatClient` 才生效，直接调用 `ChatModel` 不会读写会话记忆。
