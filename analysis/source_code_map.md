# Gemini CLI — Source Code Map (Bản đồ Module)

## 1. Tổng quan Repository

```
google-gemini/gemini-cli/
├── packages/
│   ├── core/                    # 🧠 Core engine - Logic cốt lõi của agent
│   ├── cli/                     # 🖥️ CLI interface - Terminal UI (Ink/React)
│   ├── a2a-server/              # 🌐 Agent-to-Agent server
│   ├── vscode-ide-companion/    # 📝 VS Code extension companion
│   └── test-utils/              # 🧪 Shared test utilities
├── scripts/                     # 🔧 Build, test, release scripts
├── schemas/                     # 📋 Settings JSON schema
└── third_party/                 # 📦 Third-party tools (ripgrep downloader)
```

---

## 2. Core Package — Bản đồ Chi Tiết

### 2.1 Cấu trúc thư mục

```
packages/core/src/
├── core/                        # ⭐ QUAN TRỌNG NHẤT — Vòng lặp agent chính
├── tools/                       # 🔧 Tool definitions & registry
├── agents/                      # 🤖 Sub-agent system
├── scheduler/                   # 📅 Tool scheduling & execution
├── services/                    # 🏗️ Supporting services
├── config/                      # ⚙️ Configuration management
├── prompts/                     # 📝 System prompt generation
├── hooks/                       # 🪝 Hook system (before/after events)
├── policy/                      # 🛡️ Security policy engine
├── confirmation-bus/            # 💬 Message bus for tool confirmations
├── routing/                     # 🔀 Model routing strategy
├── fallback/                    # 🔄 Fallback handling
├── availability/                # 📊 Model availability tracking
├── telemetry/                   # 📈 Telemetry & logging
├── ide/                         # 💻 IDE integration context
├── code_assist/                 # 🔑 Google OAuth/CodeAssist integration
└── utils/                       # 🛠️ Utility functions
```

---

## 3. Module Quan Trọng — Chi Tiết

### 3.1 `core/` — Trái Tim của Agent

| File | Vai trò | Mức quan trọng |
|------|---------|----------------|
| **`client.ts`** | `GeminiClient` — Orchestrator chính. Quản lý vòng lặp agent, gọi model, xử lý tool calls, next-speaker check, hook integration. | ⭐⭐⭐⭐⭐ |
| **`turn.ts`** | `Turn` class — Quản lý **một lượt** tương tác với model. Parse response stream thành events (Content, ToolCallRequest, Thought, Error, Finished). | ⭐⭐⭐⭐⭐ |
| **`geminiChat.ts`** | `GeminiChat` — Quản lý chat session, history, retry logic, stream validation, thought signatures. | ⭐⭐⭐⭐⭐ |
| **`contentGenerator.ts`** | `ContentGenerator` interface — Abstract layer cho API calls (GoogleGenAI, CodeAssist, Fake). | ⭐⭐⭐⭐ |
| **`prompts.ts`** | Entry point cho system prompt generation (delegate sang `PromptProvider`). | ⭐⭐⭐ |
| **`coreToolScheduler.ts`** | `CoreToolScheduler` — Legacy scheduler cho tool execution (policy, confirmation, execution pipeline). | ⭐⭐⭐⭐ |
| **`coreToolHookTriggers.ts`** | `executeToolWithHooks()` — Wrapper thực thi tool với Before/After hook integration. | ⭐⭐⭐⭐ |
| **`baseLlmClient.ts`** | `BaseLlmClient` — Client cho side LLM calls (next-speaker check, loop detection). | ⭐⭐⭐ |
| **`tokenLimits.ts`** | Token limit configuration per model. | ⭐⭐ |
| **`logger.ts`** | Internal logger. | ⭐ |

#### Luồng gọi chính trong `core/`:
```
GeminiClient.sendMessageStream()
  └── GeminiClient.processTurn()
        ├── Turn.run()
        │     └── GeminiChat.sendMessageStream()
        │           └── ContentGenerator.generateContentStream()
        ├── LoopDetectionService.turnStarted()
        ├── LoopDetectionService.addAndCheck()
        └── checkNextSpeaker() (via BaseLlmClient)
```

### 3.2 `tools/` — Tool Definitions

| File | Tool Name | Loại | Mô tả |
|------|-----------|------|--------|
| **`tools.ts`** | — | Base | Base classes: `BaseDeclarativeTool`, `BaseToolInvocation`, `Kind` enum | 
| **`tool-registry.ts`** | — | Registry | `ToolRegistry` — Đăng ký, truy vấn, quản lý tools |
| **`tool-names.ts`** | — | Constants | Tên tool constants, legacy aliases, plan mode tool list |
| **`read-file.ts`** | `read_file` | Read | Đọc file với line numbers |
| **`read-many-files.ts`** | `read_many_files` | Read | Đọc nhiều files cùng lúc |
| **`write-file.ts`** | `write_file` | Write | Tạo/ghi đè file |
| **`edit.ts`** | `replace` | Write | Chỉnh sửa file (find & replace) |
| **`shell.ts`** | `run_shell_command` | Execute | Chạy shell command (background support) |
| **`grep.ts`** | `grep_search` | Search | Tìm kiếm nội dung file |
| **`ripGrep.ts`** | — | Search | ripgrep integration |
| **`glob.ts`** | `glob` | Search | Tìm file theo pattern |
| **`ls.ts`** | `list_directory` | Read | Liệt kê thư mục |
| **`web-search.ts`** | `google_web_search` | Search | Tìm kiếm Google |
| **`web-fetch.ts`** | `web_fetch` | Read | Fetch nội dung web |
| **`ask-user.ts`** | `ask_user` | Interactive | Hỏi người dùng |
| **`write-todos.ts`** | `write_todos` | Write | Ghi task list |
| **`memoryTool.ts`** | `save_memory` | Write | Lưu memory vào GEMINI.md |
| **`enter-plan-mode.ts`** | `enter_plan_mode` | Plan | Vào chế độ lập kế hoạch |
| **`exit-plan-mode.ts`** | `exit_plan_mode` | Plan | Thoát chế độ lập kế hoạch |
| **`activate-skill.ts`** | `activate_skill` | Agent | Kích hoạt skill chuyên biệt |
| **`get-internal-docs.ts`** | `get_internal_docs` | Read | Đọc tài liệu nội bộ |
| **`mcp-tool.ts`** | `*__*` | MCP | MCP tool wrapper |
| **`mcp-client.ts`** | — | MCP | MCP client connection |
| **`mcp-client-manager.ts`** | — | MCP | MCP server lifecycle management |
| **`modifiable-tool.ts`** | — | Base | Base cho tools có thể sửa đổi params |
| **`tool-error.ts`** | — | Error | Tool error types |
| **`diffOptions.ts`** | — | Util | Diff generation for edit tool |
| **`constants.ts`** | — | Constants | Tool-related constants |

### 3.3 `agents/` — Sub-agent System

| File | Vai trò | Mức quan trọng |
|------|---------|----------------|
| **`local-executor.ts`** | `LocalAgentExecutor` — Chạy sub-agent trong local loop. Quản lý timeout, max turns, recovery, complete_task protocol. | ⭐⭐⭐⭐⭐ |
| **`codebase-investigator.ts`** | `CodebaseInvestigatorAgent` — Sub-agent chuyên phân tích codebase (read-only tools, structured JSON output). | ⭐⭐⭐⭐ |
| **`generalist-agent.ts`** | `GeneralistAgent` — Sub-agent đa năng với tất cả tools. | ⭐⭐⭐ |
| **`cli-help-agent.ts`** | Agent trợ giúp CLI. | ⭐⭐ |
| **`registry.ts`** | `AgentRegistry` — Đăng ký và quản lý agents. | ⭐⭐⭐ |
| **`types.ts`** | Agent type definitions (`LocalAgentDefinition`, `AgentTerminateMode`, etc.). | ⭐⭐⭐ |
| **`agent-scheduler.ts`** | `scheduleAgentTools()` — Schedule tool execution cho sub-agents. | ⭐⭐⭐ |
| **`subagent-tool.ts`** | Wraps agent thành tool để main agent có thể gọi. | ⭐⭐⭐ |
| **`subagent-tool-wrapper.ts`** | Wrapper layer cho subagent tools. | ⭐⭐ |
| **`local-invocation.ts`** | Local agent invocation logic. | ⭐⭐ |
| **`remote-invocation.ts`** | Remote agent invocation (A2A protocol). | ⭐⭐ |
| **`a2a-client-manager.ts`** | A2A client connection management. | ⭐⭐ |
| **`agentLoader.ts`** | Dynamic agent loading from configuration. | ⭐⭐ |
| **`acknowledgedAgents.ts`** | User acknowledgment tracking for agents. | ⭐ |
| **`utils.ts`** | Template string utility for agent prompts. | ⭐ |

### 3.4 `scheduler/` — Tool Scheduling (New Architecture)

| File | Vai trò | Mức quan trọng |
|------|---------|----------------|
| **`scheduler.ts`** | `Scheduler` — New event-driven tool orchestrator. Handles batch processing, queuing, confirmation loop. | ⭐⭐⭐⭐ |
| **`types.ts`** | Type definitions cho tool call lifecycle (`ValidatingToolCall`, `ExecutingToolCall`, `CompletedToolCall`, etc.). | ⭐⭐⭐⭐ |
| **`tool-executor.ts`** | `ToolExecutor` — Thực thi tool thực tế (with hooks, output streaming). | ⭐⭐⭐⭐ |
| **`state-manager.ts`** | `SchedulerStateManager` — Quản lý trạng thái tool calls. | ⭐⭐⭐ |
| **`confirmation.ts`** | `resolveConfirmation()` — Confirmation loop logic (ask user, modify, retry). | ⭐⭐⭐ |
| **`policy.ts`** | `checkPolicy()`, `updatePolicy()` — Policy check and update for tools. | ⭐⭐⭐ |
| **`tool-modifier.ts`** | `ToolModificationHandler` — Handle "Modify with Editor" flow. | ⭐⭐ |

### 3.5 `services/` — Supporting Services

| File | Vai trò | Mức quan trọng |
|------|---------|----------------|
| **`loopDetectionService.ts`** | `LoopDetectionService` — 3-level loop detection (tool call, content chanting, LLM-based). | ⭐⭐⭐⭐⭐ |
| **`chatCompressionService.ts`** | `ChatCompressionService` — Nén lịch sử chat khi context window gần đầy. | ⭐⭐⭐⭐ |
| **`chatRecordingService.ts`** | `ChatRecordingService` — Ghi lại toàn bộ conversation để export/resume. | ⭐⭐⭐ |
| **`toolOutputMaskingService.ts`** | `ToolOutputMaskingService` — Ẩn tool output cũ để tiết kiệm context. | ⭐⭐⭐ |
| **`modelConfigService.ts`** | `ModelConfigService` — Quản lý model configuration, aliases, overrides. | ⭐⭐⭐ |

### 3.6 `prompts/` — System Prompt Generation

| File | Vai trò | Mức quan trọng |
|------|---------|----------------|
| **`promptProvider.ts`** | `PromptProvider` — Orchestrates prompt generation, section composition. | ⭐⭐⭐⭐ |
| **`snippets.ts`** | Prompt snippets cho Gemini 3 (preview models). Chứa toàn bộ system prompt logic. | ⭐⭐⭐⭐⭐ |
| **`snippets.legacy.ts`** | Prompt snippets cho Gemini 2 models. | ⭐⭐⭐ |
| **`utils.ts`** | Prompt utilities (env var resolution, substitutions). | ⭐⭐ |

### 3.7 `hooks/` — Hook System

| File | Vai trò |
|------|---------|
| **`types.ts`** | Hook type definitions, output classes. |
| Và các file liên quan | BeforeAgent, AfterAgent, BeforeModel, AfterModel, BeforeTool, AfterTool, BeforeToolSelection. |

### 3.8 `utils/` — Quan trọng nhất

| File | Vai trò | Mức quan trọng |
|------|---------|----------------|
| **`nextSpeakerChecker.ts`** | `checkNextSpeaker()` — LLM-based check xem agent nên tiếp tục hay dừng. | ⭐⭐⭐⭐⭐ |
| **`thoughtUtils.ts`** | Parse "thought" content từ model response. | ⭐⭐⭐ |
| **`partUtils.ts`** | Utility cho xử lý `Part` objects (text extraction). | ⭐⭐⭐ |
| **`tokenCalculation.ts`** | Token counting (sync estimate + API countTokens). | ⭐⭐⭐ |
| **`summarizer.ts`** | Chat history summarization. | ⭐⭐⭐ |
| **`planUtils.ts`** | Plan file validation (path safety, content check). | ⭐⭐⭐ |
| **`retry.ts`** | `retryWithBackoff()` — Retry logic with exponential backoff. | ⭐⭐⭐ |
| **`errors.ts`** | Error classification and friendly messages. | ⭐⭐ |
| **`security.ts`** | Security utilities. | ⭐⭐ |
| **`environmentContext.ts`** | Directory context, initial chat history. | ⭐⭐ |
| **`events.ts`** | Core event emitter. | ⭐⭐ |
| **`toolCallContext.ts`** | AsyncLocalStorage for tool call context tracking. | ⭐⭐ |
| **`promptIdContext.ts`** | AsyncLocalStorage for prompt ID context. | ⭐⭐ |
| **`tool-utils.ts`** | Tool suggestion (fuzzy match for typos). | ⭐ |
| **`debugLogger.ts`** | Debug logging utility. | ⭐ |

---

## 4. CLI Package — UI Layer

```
packages/cli/src/
├── ui/
│   ├── app.tsx                  # Root React/Ink component
│   ├── components/              # UI components (chat, tools, status)
│   ├── themes/                  # Color themes (dracula, github-light, etc.)
│   └── utils/                   # UI utilities
├── utils/
│   ├── commands.ts              # Slash command handlers (/help, /bug, etc.)
│   ├── sandbox.ts               # Sandbox configuration
│   ├── sessions.ts              # Session management
│   ├── hookUtils.ts             # Hook configuration loading
│   ├── agentSettings.ts         # Agent configuration loading
│   └── ...
├── nonInteractiveCli.ts         # Non-interactive mode entry point
└── index.ts                     # CLI entry point
```

---

## 5. Dependency Graph — Module Relationships

```
┌─────────────────────────────────────────────────────────┐
│                     CLI Layer                            │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │ Terminal  │  │ Non-         │  │ Commands         │  │
│  │ UI (Ink)  │  │ Interactive  │  │ (/help, /model)  │  │
│  └─────┬────┘  └──────┬───────┘  └────────┬─────────┘  │
│        │               │                   │            │
└────────┼───────────────┼───────────────────┼────────────┘
         │               │                   │
         ▼               ▼                   ▼
┌─────────────────────────────────────────────────────────┐
│                    Core Layer                            │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              GeminiClient (Orchestrator)           │   │
│  │  ┌──────────┐ ┌──────────┐ ┌─────────────────┐  │   │
│  │  │processTurn│ │sendMsg   │ │FireHooks        │  │   │
│  │  │          │ │Stream    │ │(Before/After)   │  │   │
│  │  └────┬─────┘ └────┬─────┘ └────────┬────────┘  │   │
│  └───────┼─────────────┼────────────────┼───────────┘   │
│          │             │                │               │
│          ▼             │                │               │
│  ┌───────────────┐     │                │               │
│  │ Turn.run()    │◄────┘                │               │
│  │ (Parse stream)│                      │               │
│  └───────┬───────┘                      │               │
│          │                              │               │
│          ▼                              │               │
│  ┌────────────────┐                     │               │
│  │ GeminiChat     │                     │               │
│  │ (History mgmt) │                     │               │
│  └───────┬────────┘                     │               │
│          │                              │               │
│          ▼                              │               │
│  ┌────────────────────┐                 │               │
│  │ ContentGenerator   │                 │               │
│  │ (GoogleGenAI SDK)  │                 │               │
│  └────────────────────┘                 │               │
│                                         │               │
│  ┌──────────────────┐   ┌───────────────▼──────────┐   │
│  │ ToolRegistry     │   │ CoreToolScheduler        │   │
│  │ ┌──────────────┐ │   │ ┌──────────┐┌──────────┐│   │
│  │ │Built-in Tools│ │◄──│ │PolicyEng ││ToolExec  ││   │
│  │ │MCP Tools     │ │   │ └──────────┘└──────────┘│   │
│  │ │Discovered    │ │   └──────────────────────────┘   │
│  │ └──────────────┘ │                                  │
│  └──────────────────┘                                  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Services                              │   │
│  │ ┌──────────┐ ┌──────────┐ ┌─────────────────┐   │   │
│  │ │LoopDet   │ │ChatCompr │ │ModelRouter     │   │   │
│  │ │Service   │ │Service   │ │Service         │   │   │
│  │ └──────────┘ └──────────┘ └─────────────────┘   │   │
│  └──────────────────────────────────────────────────┘   │
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Agent System                          │   │
│  │ ┌──────────────┐ ┌───────────────────────────┐   │   │
│  │ │LocalAgent    │ │CodebaseInvestigator      │   │   │
│  │ │Executor      │ │GeneralistAgent           │   │   │
│  │ └──────────────┘ └───────────────────────────┘   │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

---

## 6. Entry Points — Điểm Khởi Đầu

### 6.1 Interactive Mode
```
packages/cli/src/index.ts
  → app.tsx (Ink/React render)
  → useGemini hook
  → GeminiClient.sendMessageStream()
```

### 6.2 Non-Interactive Mode
```
packages/cli/src/nonInteractiveCli.ts
  → GeminiClient.sendMessageStream()
  → Process all events
  → Exit with result
```

### 6.3 A2A Server
```
packages/a2a-server/
  → HTTP server
  → Agent-to-Agent protocol
  → GeminiClient (core)
```

---

## 7. Tóm tắt: Các File Quan Trọng Nhất Cần Đọc

Nếu muốn hiểu Gemini CLI nhanh nhất, hãy đọc theo thứ tự:

1. **`packages/core/src/core/client.ts`** — Orchestrator chính, vòng lặp agent.
2. **`packages/core/src/core/turn.ts`** — Quản lý một lượt tương tác model.
3. **`packages/core/src/core/geminiChat.ts`** — Chat session, history, retry.
4. **`packages/core/src/prompts/snippets.ts`** — System prompt (chứa "linh hồn" của agent).
5. **`packages/core/src/core/coreToolScheduler.ts`** — Tool scheduling pipeline.
6. **`packages/core/src/services/loopDetectionService.ts`** — Loop detection 3 cấp.
7. **`packages/core/src/utils/nextSpeakerChecker.ts`** — Self-reflection mechanism.
8. **`packages/core/src/agents/local-executor.ts`** — Sub-agent execution.
9. **`packages/core/src/tools/enter-plan-mode.ts`** + **`exit-plan-mode.ts`** — Plan Mode.
10. **`packages/core/src/core/coreToolHookTriggers.ts`** — Hook integration.
