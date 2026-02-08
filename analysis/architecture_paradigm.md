# Gemini CLI — Architecture & Agent Paradigm Analysis

## Tóm tắt (Executive Summary)

Gemini CLI (`google-gemini/gemini-cli`) là một **Hybrid ReAct Agent** với các tính năng mở rộng bao gồm **Plan Mode** (lập kế hoạch trước khi hành động), **Loop Detection** (phát hiện vòng lặp vô hạn), **Next-Speaker Checker** (tự kiểm tra xem agent có nên tiếp tục hay dừng), và **Sub-agent Architecture** (ủy quyền cho các agent con chuyên biệt). Nó **không phải CodeAct thuần túy** — agent không sinh code rồi tự chạy; thay vào đó, nó sử dụng các tool declarations (function calling) đã được định nghĩa sẵn theo mô hình ReAct.

---

## 1. Xác định Mô hình Agent: ReAct + Extensions

### 1.1 Tại sao là ReAct?

Gemini CLI hoạt động theo vòng lặp **Reason → Act → Observe** kinh điển của ReAct:

1. **Reason (Suy luận):** Model nhận prompt, lịch sử hội thoại, và suy luận (bao gồm cả "thoughts" nếu dùng preview model).
2. **Act (Hành động):** Model trả về `functionCall` — tức là yêu cầu gọi một tool cụ thể với tham số cụ thể.
3. **Observe (Quan sát):** Kết quả của tool được đưa trở lại model dưới dạng `functionResponse`, và vòng lặp tiếp tục.

**Bằng chứng từ mã nguồn:**

Trong `packages/core/src/core/client.ts`, hàm `processTurn()` thể hiện rõ vòng lặp ReAct:

```typescript
// packages/core/src/core/client.ts, line ~656-681
const resultStream = turn.run(modelConfigKey, request, linkedSignal, displayContent);
for await (const event of resultStream) {
  // ... process events including ToolCallRequest, Content, Error ...
  yield event;
}
```

Trong `packages/core/src/core/turn.ts`, class `Turn` xử lý một lượt tương tác:

```typescript
// packages/core/src/core/turn.ts, line ~300-325
const parts = resp.candidates?.[0]?.content?.parts ?? [];
for (const part of parts) {
  if (part.thought) {
    yield { type: GeminiEventType.Thought, value: thought, traceId };
  }
}
const text = getResponseText(resp);
if (text) {
  yield { type: GeminiEventType.Content, value: text, traceId };
}
const functionCalls = resp.functionCalls ?? [];
for (const fnCall of functionCalls) {
  const event = this.handlePendingFunctionCall(fnCall, traceId);
  if (event) yield event;
}
```

### 1.2 Tại sao KHÔNG phải CodeAct?

CodeAct agents sinh code (thường là Python) rồi tự thực thi trong một sandbox. Gemini CLI **không làm điều này**. Thay vào đó:

- Agent sử dụng **pre-defined tools** (function declarations) được đăng ký trong `ToolRegistry`.
- Mỗi tool có schema rõ ràng, với tham số được validate bởi `BaseDeclarativeTool.build()`.
- Agent không tự sinh code để chạy; nó gọi các tool API như `read_file`, `write_file`, `run_shell_command`, `grep_search`, v.v.

**Bằng chứng:**

```typescript
// packages/core/src/tools/tool-names.ts
export const GLOB_TOOL_NAME = 'glob';
export const SHELL_TOOL_NAME = 'run_shell_command';
export const GREP_TOOL_NAME = 'grep_search';
export const READ_FILE_TOOL_NAME = 'read_file';
export const WRITE_FILE_TOOL_NAME = 'write_file';
export const EDIT_TOOL_NAME = 'replace';
// ... và nhiều tool khác
```

### 1.3 Các tính năng mở rộng vượt ra ngoài ReAct cơ bản

Gemini CLI không chỉ là ReAct đơn giản. Nó bổ sung nhiều cơ chế tinh vi:

| Tính năng | Mô tả | Vị trí trong code |
|-----------|--------|-------------------|
| **Plan Mode** | Chế độ lập kế hoạch read-only trước khi thực thi | `tools/enter-plan-mode.ts`, `tools/exit-plan-mode.ts` |
| **Loop Detection** | Phát hiện agent bị lặp vô hạn (3 cấp) | `services/loopDetectionService.ts` |
| **Next-Speaker Check** | LLM tự kiểm tra xem nên tiếp tục hay dừng | `utils/nextSpeakerChecker.ts` |
| **Sub-agents** | Agent con chuyên biệt (Codebase Investigator) | `agents/codebase-investigator.ts`, `agents/local-executor.ts` |
| **Chat Compression** | Tự động nén lịch sử khi context window đầy | `services/chatCompressionService.ts` |
| **Hook System** | Hệ thống hook trước/sau agent, model, tool | `hooks/types.ts`, `core/coreToolHookTriggers.ts` |
| **Model Routing** | Tự động chọn model phù hợp cho từng turn | `routing/routingStrategy.ts` |

---

## 2. Planning Module (Lập kế hoạch)

### 2.1 Plan Mode — Chế độ Lập Kế Hoạch Chuyên Dụng

Gemini CLI có một **Plan Mode** chính thức, được kích hoạt bằng tool `enter_plan_mode` và kết thúc bằng `exit_plan_mode`.

**Cơ chế hoạt động:**

1. **Enter Plan Mode:** Agent gọi `enter_plan_mode` → `Config.setApprovalMode(ApprovalMode.PLAN)`.
2. **Chỉ đọc:** Trong Plan Mode, agent chỉ được phép sử dụng read-only tools:
   - `glob`, `grep_search`, `read_file`, `list_directory`, `google_web_search`, `ask_user`, `exit_plan_mode`
   - Plus: `write_file` và `replace` chỉ trong thư mục plans.
3. **4 Phases:** Requirements Understanding → Project Exploration → Design & Planning → Review & Approval.
4. **Exit Plan Mode:** Agent gọi `exit_plan_mode` với đường dẫn đến file kế hoạch → User duyệt → Agent chuyển sang chế độ thực thi.

**Bằng chứng từ mã nguồn:**

```typescript
// packages/core/src/tools/enter-plan-mode.ts, line ~116-132
async execute(_signal: AbortSignal): Promise<ToolResult> {
  if (this.confirmationOutcome === ToolConfirmationOutcome.Cancel) {
    return { llmContent: 'User cancelled entering Plan Mode.', returnDisplay: 'Cancelled' };
  }
  this.config.setApprovalMode(ApprovalMode.PLAN);
  return { llmContent: 'Switching to Plan mode.', returnDisplay: '...' };
}
```

```typescript
// packages/core/src/tools/exit-plan-mode.ts, line ~207-262
async execute(_signal: AbortSignal): Promise<ToolResult> {
  // ... validation ...
  if (payload?.approved) {
    const newMode = payload.approvalMode ?? ApprovalMode.DEFAULT;
    this.config.setApprovalMode(newMode);
    this.config.setApprovedPlanPath(resolvedPlanPath);
    return {
      llmContent: `Plan approved. Switching to ${description}...`,
    };
  }
}
```

### 2.2 System Prompt Lifecycle: Research → Strategy → Execution

Ngay cả khi không ở Plan Mode, system prompt yêu cầu agent tuân theo lifecycle:

```
Research → Strategy → Execution (Plan → Act → Validate)
```

**Bằng chứng:**

```typescript
// packages/core/src/prompts/snippets.ts, line ~213-226
return `
# Primary Workflows
## Development Lifecycle
Operate using a **Research -> Strategy -> Execution** lifecycle.
For the Execution phase, resolve each sub-task through an iterative **Plan -> Act -> Validate** cycle.

1. **Research:** Systematically map the codebase and validate assumptions...
2. **Strategy:** Formulate a grounded plan based on your research...
3. **Execution:** For each sub-task:
   - **Plan:** Define the specific implementation approach...
   - **Act:** Apply targeted, surgical changes...
   - **Validate:** Run tests and workspace standards...
`;
```

### 2.3 Sub-agent Planning: Codebase Investigator

Agent `codebase_investigator` là sub-agent chuyên về phân tích codebase **trước khi thực thi**. Nó chỉ dùng read-only tools và trả về báo cáo có cấu trúc.

```typescript
// packages/core/src/agents/codebase-investigator.ts, line ~117-124
toolConfig: {
  tools: [LS_TOOL_NAME, READ_FILE_TOOL_NAME, GLOB_TOOL_NAME, GREP_TOOL_NAME],
},
```

---

## 3. Verification / Reflection Module (Tự kiểm tra)

### 3.1 Next-Speaker Checker — LLM Tự Kiểm Tra

Sau mỗi turn không có tool call, agent sử dụng **một LLM call riêng biệt** để kiểm tra xem nó nên tiếp tục hay dừng. Đây là cơ chế **self-reflection** quan trọng.

**Bằng chứng:**

```typescript
// packages/core/src/core/client.ts, line ~728-758
if (!turn.pendingToolCalls.length && signal && !signal.aborted) {
  const nextSpeakerCheck = await checkNextSpeaker(
    this.getChat(), this.config.getBaseLlmClient(), signal, prompt_id
  );
  if (nextSpeakerCheck?.next_speaker === 'model') {
    const nextRequest = [{ text: 'Please continue.' }];
    turn = yield* this.sendMessageStream(nextRequest, signal, prompt_id, boundedTurns - 1, false);
    return turn;
  }
}
```

```typescript
// packages/core/src/utils/nextSpeakerChecker.ts, line ~13-17
const CHECK_PROMPT = `Analyze *only* the content and structure of your immediately preceding response...
**Decision Rules (apply in order):**
1. **Model Continues:** If your last response explicitly states an immediate next action...
2. **Question to User:** If your last response ends with a direct question...
3. **Waiting for User:** If your last response completed a thought...`;
```

### 3.2 Loop Detection — 3 Cấp Phát Hiện Vòng Lặp

`LoopDetectionService` sử dụng **3 cấp phát hiện** từ đơn giản đến phức tạp:

| Cấp | Cơ chế | Ngưỡng |
|-----|--------|--------|
| **1. Tool Call Loop** | Hash tool name + args, đếm lặp liên tiếp | ≥ 5 lần giống hệt nhau |
| **2. Content Chanting** | Sliding window hash trên text output | ≥ 10 chunks giống nhau gần nhau |
| **3. LLM-based Detection** | Dùng LLM để phân tích lịch sử conversation | Confidence ≥ 0.9, double-check bằng model thứ 2 |

**Bằng chứng:**

```typescript
// packages/core/src/services/loopDetectionService.ts, line ~29-31
const TOOL_CALL_LOOP_THRESHOLD = 5;
const CONTENT_LOOP_THRESHOLD = 10;
const CONTENT_CHUNK_SIZE = 50;
```

```typescript
// packages/core/src/services/loopDetectionService.ts, line ~68-77
const LOOP_DETECTION_SYSTEM_PROMPT = `You are a sophisticated AI diagnostic agent 
specializing in identifying when a conversational AI is stuck in an unproductive state...`;
```

### 3.3 Validation trong System Prompt

System prompt yêu cầu agent **luôn validate** sau khi thực thi:

> **Validation is the only path to finality.** Never assume success or settle for unverified changes. Rigorous, exhaustive verification is mandatory...

```typescript
// packages/core/src/prompts/snippets.ts, line ~223
//    - **Validate:** Run tests and workspace standards to confirm the success...
```

### 3.4 Hook System — External Verification

Hook system cho phép kiểm tra trước/sau mỗi agent turn, model call, và tool call:

- `BeforeAgent` / `AfterAgent`: Kiểm tra trước/sau toàn bộ agent interaction.
- `BeforeModel` / `AfterModel`: Kiểm tra trước/sau mỗi LLM call.
- `BeforeTool` / `AfterTool`: Kiểm tra trước/sau mỗi tool execution.

**Bằng chứng:**

```typescript
// packages/core/src/core/coreToolHookTriggers.ts, line ~66-148
export async function executeToolWithHooks(
  invocation, toolName, signal, tool, ...
): Promise<ToolResult> {
  const hookSystem = config?.getHookSystem();
  if (hookSystem) {
    const beforeOutput = await hookSystem.fireBeforeToolEvent(toolName, toolInput, mcpContext);
    if (beforeOutput?.shouldStopExecution()) { /* stop */ }
    if (beforeOutput?.getBlockingError()) { /* block */ }
  }
  // Execute actual tool...
  if (hookSystem) {
    const afterOutput = await hookSystem.fireAfterToolEvent(...);
    // ... check stop/block/additional context
  }
}
```

---

## 4. So sánh với các Agent Framework phổ biến

| Đặc điểm | ReAct (gốc) | CodeAct | Gemini CLI |
|-----------|-------------|---------|------------|
| Vòng lặp cốt lõi | Reason→Act→Observe | Generate Code→Execute→Observe | Reason→Act→Observe |
| Tool sử dụng | Pre-defined tools | Code generation | Pre-defined tools |
| Planning | Không | Không | **Có (Plan Mode)** |
| Self-reflection | Không | Không | **Có (Next-Speaker, Loop Detection)** |
| Sub-agents | Không | Không | **Có (Codebase Investigator)** |
| Context management | Không | Không | **Có (Compression, Masking)** |
| Hook system | Không | Không | **Có (Before/After hooks)** |

---

## 5. Kết luận

Gemini CLI là một **Enhanced ReAct Agent** — lấy nền tảng là mô hình ReAct kinh điển nhưng bổ sung các cơ chế tinh vi:

1. **Planning trước khi hành động** (Plan Mode với 4 phases).
2. **Self-verification liên tục** (Next-Speaker Checker, Loop Detection 3 cấp).
3. **Sub-agent delegation** (Codebase Investigator cho phân tích chuyên sâu).
4. **Context window management** (Chat Compression, Tool Output Masking).
5. **Extensible hook system** (trước/sau mỗi bước trong pipeline).

Mô hình gần nhất để mô tả kiến trúc này là: **ReAct + Plan-Act-Validate lifecycle + Multi-level Reflection + Hierarchical Agent System**.
