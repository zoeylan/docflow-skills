---
name: docflow-contract-audit
description: "DocFlow 合同审核 Agent 端到端自动化工作流，包含一审，复审流程。触发词：合同审核、contract audit"
---

## 工具使用约束（重要）

**禁止使用以下工具：**

- ❌ `start_code_interpreter_session` 及任何 code interpreter 相关工具
- ❌ 用 Read 工具直接读取 `.docx` / `.pdf` 等合同文件内容
- ❌ 用 base64 解码或 Python 脚本解析合同文件

**只能使用：**

- ✅ `start_browser_session` 和其它 `browser_*` 工具（DocFlow 平台操作）
- ✅ Bash + curl（API 调用、文件上传）

合同内容必须由 DocFlow 平台分析，不要自行读取或解析合同文件。

## 浏览器使用规则

**默认情况下，整个对话只创建一次 browser session**。

1. 第一次需要浏览器时调用 `start_browser_session` 一次（参数：`{"region":"ap-northeast-1","timeout_seconds":28800}`），记住返回的 session_id
2. 后续所有 browser 操作（navigate、click、type、snapshot 等）都用同一个 session_id
3. **不要**因为 snapshot 看到非预期内容、页面停在旧地方就重启 session——只是页面没刷新而已，用 `browser_navigate` 到目标 URL 即可
4. **不要**调用 `stop_browser_session`

### 浏览器工具失败的处理

如果 browser_* 工具失败：
1. 用 Bash `sleep 10` 等 10 秒（让 MCP WebSocket 自动恢复）
2. 用同一个 session_id **重试**同一个工具

如果重试 2 次后仍然报 "No connection for session"、"SessionNotFound"、"WebSocket disconnected" 等连接性错误：
- 说明底层 MCP 连接真的丢了，**这种情况下可以**重新调用 `start_browser_session`（参数同上）
- 拿到新 session_id 后，重新登录并 navigate 到之前的页面
- 前端会平滑过渡到新画面（不会黑屏）

**重要：恢复过程对用户必须完全静默。** 不要在回复用户的文字中出现 "MCP 重连"、"WebSocket 断开"、"session 过期"、"重新启动浏览器"、"重新连接" 这类技术内部细节。直接做完恢复继续业务流程即可，用户只关心合同审核进度。

## 用户文件位置（重要）

用户上传的附件已经在容器的 `/workspace/` 目录下，文件名就是用户上传时的原始文件名（系统消息会告诉你具体文件名）。**直接用 Bash + curl 上传，不要用 code interpreter，不要用 Glob 搜索。**

## 本 Agent 中约定的术语如下表

| 术语 | 说明 | 进一步澄清 |
|---|---|---|
| BASE_URL | https://docflow.textin.com/agent-center | DocFlow 平台地址 |
| 用户 | 操作 Super Agent 的使用者 | 包括通过任何 channel 连接到 Super Agent 的使用者，包括 Web UI、以及 feishu 等 IM 工具 |
| case_no | 文档上传到 DocFlow 平台后，会获得的唯一识别码 | 该值会在调用 DocFlow 平台的上传文档 api 后获得，与 api 返回结果中的变量名一致 |

## 响应输出约定

### 语言一致性（最高优先级）

**所有回复的语言必须与用户当前消息的语言保持一致，全流程不切换。**

- 用户用中文 → 全程用中文回复（包括进度提示、决策建议、最终结果）
- 用户用英文 → 全程用英文回复
- 用户用其他语言（日语 / 韩语 / 西语等）→ 全程用同语言回复
- 同一对话内即使中途用户没再发新消息，也以**最近一次用户消息的语言**为准
- 即使引用合同条款、字段名、技术术语原文，包裹这些原文的解释和过渡文字也要用用户的语言

不要混合语言。不要因为参考文档是英文就突然切换。不要在中文回复里掺英文短语（除非是通用专有名词如 "OA"、"CASE-XXX"、"DocFlow"）。

### 输出原则：只讲业务进度，不讲技术细节

通过 channel 输出给用户的文字内容**只包含业务层面的进展**。技术细节（特别是底层连接、session 管理）必须**完全静默**。

**✅ 应该输出（业务进度）：**
- "附件已确认：采购合同v1.docx"
- "已登录 DocFlow 平台"
- "文件上传成功，案件已创建：CASE-0602-XXXXX"
- "工作流已启动，正在文档解析..."
- "文档分类完成"
- "字段提取完成（21 项）"
- "规则校验完成：21 通过 / 1 失败"
- "风险识别完成：高风险（违约责任上限缺失）"
- "审核决策建议：驳回 — 第 11.4 条无限补足敞口对甲方风险过高"
- "审核已提交：驳回"
- "OA 系统审批已完成"

**❌ 绝不输出（技术细节）：**
- "MCP 服务重连"、"WebSocket 连接断开"、"connection lost"
- "browser session 已过期"、"重新启动浏览器"、"重新连接"
- "session_id"、tool_use_id 等任何 session ID 字符串
- 工具调用名（`start_browser_session`、`browser_navigate` 等）
- HTTP 状态码、API 错误堆栈
- "正在调用 XXX 工具"、"正在轮询" 这类 agent 内部行为

如果遇到错误必须告知用户，用业务语言描述，例如："上传文件时遇到问题，请稍后重试"，而不是 "curl returned 500"。

### 关键任务节点（必须输出业务进度）：

- **首轮审核**：
  - Step 1：附件已确认 / 平台已登录
  - Step 2：文件上传成功 / 案件已创建（输出 case_no 和 case_id）
  - Step 3：工作流各阶段进度（文档解析 / 文档分类 / 字段提取 / 规则校验 / 风险检测 完成）
- **复审**：
  - Step 1：附件已确认 / CASE_NO 已确认 / 平台已登录
  - Step 2：文件已追加上传
  - Step 3：合同比对开始 / 比对结果摘要
  - Step 4：各阶段完成进度

## 首轮审核流程

当用户没有提供 case_no 时，判定需要对用户上传的文件进行首轮审核。

### Step 1: 前置条件检查

**附件检查：**确认用户在对话过程中是否上传了附件，如果没有任何附件，需要提醒用户上传

**BASE_URL 检查：**确认 BASE_URL 是否可用

执行以下命令判断 BASE_URL 是否可用：

```
curl -sI -o /dev/null -w "%{http_code}" -L {BASE_URL}
```

如果 BASE_URL 可用，继续执行后续步骤；如果不可用，提示用户后端服务不可用。

使用浏览器登录 BASE_URL 的登录页：

```
详情页地址: ${BASE_URL}/login?redirect=%2Fagent-center%2Fcontract-audit%2Fcases
```

在登录页面中，输入用户凭证：

- 用户名或邮箱：superagent
- 密码：Superagent@2026

点击【登录】按钮，完成登录操作。

### Step 2: 上传文件创建案件

将用户上传的附件，上传到 DocFlow 平台。文件已经在容器的 `/workspace/` 目录下，直接用 Bash + curl 上传，**不要使用 code interpreter，不要用 Glob 搜索**。

例如用户上传了 `采购合同v1.docx`，直接执行：

```
curl -X POST {BASE_URL}/api/contract-audit/upload \
  -H "Authorization: Bearer pat_2J_xTUxbIV5wJaoVkYLf6TRsesoa0JS8yC381AlKrkA" \
  -F "files=@/workspace/采购合同v1.docx" \
  -F "doc_type=auto"
```

把 `采购合同v1.docx` 替换成用户实际上传的文件名（系统消息会告诉你）。

返回结构：

```json
{
  "case_id": "42",
  "case_no": "CASE-0419-XXXXX",
  "file_ids": ["101"],
  "file_count": 1,
  "mode": "new"
}  
```

**操作要点**：

- 记录返回的 `case_no`，和 case_id，后续所有步骤需要用到
- 确保文件上传成功后再进入下一步

### Step 3: 等待工作流完成

使用浏览器打开案件详情页，监控工作流执行进度。**复用 Step 1 已经创建的 browser session**，不要重新调用 `start_browser_session`。

只需要在【运行流程】Tab 下观察工作流执行进度，**不要**尝试使用其它 Tab 页观察流程

```
详情页地址: ${BASE_URL}/contract-audit/cases/${case_no}
```

**工作流阶段**：

1. **文档解析** (`doc_parse`) — 自动 OCR 识别与结构化提取
2. **文档分类** (`doc_classify`) — 自动分类
3. **字段提取** (`field_extract`) — 自动提取合同字段
4. **文档匹配** (`doc_match`) — 自动匹配
5. **规则校验** (`rule_validate`) — 自动校验
6. **风险检测** (`risk_detect`) — 自动检测风险
7. **审核决策** (`audit_decide`) — **需要人工确认**
8. **系统交互** (`system_interact`) — 自动执行

**等待机制**：

所有阶段均采用 **5 秒间隔轮询**：

1. 页面加载完成后，点击「运行流程」按钮启动工作流

2. 每 5 秒做一次：Bash `sleep 5` → `browser_snapshot`（不要 navigate！navigate 会重新加载页面打断工作流）

3. 在相同 channel 中通知用户 case 已经创建，case_no 是什么

4. **如何判断 audit_decide 已就绪**：在 `browser_snapshot` 返回的页面内容中查找以下任一关键词，出现即代表 audit_decide 已经就绪，**立即停止轮询**：
   - "审核决策"
   - "建议驳回"、"建议通过"、"建议挂起"
   - "待审核"
   - 出现【通过】、【驳回】、【挂起】按钮
   
   **重要**：一旦看到上述任一关键词就停止 sleep 循环，不要再继续轮询。最多轮询 20 次（约 100 秒），如果还没看到上述关键词，告知用户工作流可能卡住，并停止轮询。

5. 查看「审核决策」中给出的反馈（"建议驳回"/"建议通过"/"建议挂起"和原因），并在相同 channel 中通知用户，等待用户决策

6. 根据用户给出的反馈，执行审核决策操作：
   - **不要 navigate**！页面已经停在案件详情页，直接在当前页面操作
   - 先 `browser_snapshot` 获取页面元素，找到「审核决策」卡片中的【通过】/【驳回】/【挂起】按钮
   - 用 `browser_click` 点击对应按钮（如果页面没滚到位，用 `browser_evaluate` 执行 `document.querySelector('按钮selector').scrollIntoView()` 先滚动）
   - 在弹出的确认对话框中，用 `browser_type` 填写用户反馈的内容作为审核意见
   - `browser_click` 点击确认按钮提交决策

7. **决策提交后的判停规则（避免无限轮询）**：

   **如果用户决策是【驳回】或【挂起】**：
   - **不需要等待 `system_interact`**，提交决策后立即结束流程，进入第 8 步通知用户。
   - 驳回 / 挂起不会触发 OA 推送或后续系统交互，继续轮询只会浪费时间。

   **如果用户决策是【通过】**：
   - 以 5 秒间隔轮询（`browser_snapshot`，不要 navigate），等待 `system_interact` 完成。
   - 在 snapshot 返回内容中查找以下任一关键词，出现即代表 `system_interact` 完成，**立即停止轮询**：
     - "OA 审批已推送"、"OA审批已推送"
     - "已推送至 OA"、"已推送至OA"
     - "系统交互完成"、"system_interact" 阶段卡片显示"完成"或绿色状态
   - **最多轮询 12 次（约 60 秒）**，超时后直接进入第 8 步通知用户，不要继续等待。

8. 处理完毕后，在相同 channel 中通知用户处理的最终结果，注意使用以下格式：

   ```
   合同审核结果通知：
   CASE_NO：{case_no}  
   CASE_ID：{case_id}  
   审核结论：{审核决策结论}  
   审核要点：
   - ...（总结审核理由摘要）
   ```
   
   

## 复审流程

当用户上传文档、并提供了 case_no 时，代表用户发起了对已有案件的复审流程。

### Step 1: 前置条件检查

**附件检查：**确认用户在对话过程中是否上传了附件，如果没有任何附件，需要提醒用户上传

**BASE_URL 检查：**确认 BASE_URL 是否可用

执行以下命令判断 BASE_URL 是否可用：

```
curl -sI -o /dev/null -w "%{http_code}" -L {BASE_URL}
```

如果 BASE_URL 可用，继续执行后续步骤；如果不可用，提示用户后端服务不可用。

通过浏览器工具，打开案件详情页地址：

**启动浏览器**：如果本对话已经创建过 browser session，**直接用已有的 session_id**，不要再调用 `start_browser_session`。如果还没创建，按照系统级 BROWSER SESSION RULES 创建一次。

```
${BASE_URL}/contract-audit/cases/${case_no}
```

如果进入到登录页面，输入以下凭证，然后点击【登录】按钮：

- 用户名或邮箱：superagent
- 密码：Superagent@2026

确认进入到案件详情页，继续执行后续操作。

### Step 2: Agent 助手追加文件并重跑任务

**操作步骤**：

1. 在浏览器页面左侧，找到「 Agent 助手」聊天区域

2. 将用户在本次聊天过程中上传的文件上传到 DocFlow 平台。文件已经在容器的 `/workspace/` 目录下，直接用 Bash + curl 上传，**不要使用 code interpreter**：

   ```
   curl -X POST {BASE_URL}/api/contract-audit/upload \
     -H "Authorization: Bearer pat_2J_xTUxbIV5wJaoVkYLf6TRsesoa0JS8yC381AlKrkA" \
     -F "files=@/workspace/{文件名}" \
     -F "doc_type=auto" \
     -F "case_id={case_id}" \
     -F "mode=append"
   ```

   把 `{文件名}` 替换成用户实际上传的文件名（系统消息会告诉你）。

3. 在「 Agent 助手」的聊天窗口中发送消息：`追加文件,重跑任务`

4. Agent 助手会自动追加文件并触发 Pipeline 重跑

**操作要点**：

- 确认 Agent 助手已开始处理后进入下一步

### Step 3: 等待合同比对完成，预览比对结果

以 5 秒间隔轮询，继续在浏览器中等待合同对比阶段完成，并预览比对结果。

**等待 `doc_compare` 完成**：

等待页面中 `doc_compare` 阶段卡片状态变为完成（ <上一次上传文档> 与 <本次上传文档> 差异对比）。

**预览比对**：

1. 在 `doc_compare` 阶段卡片中，找到并点击「预览比对」按钮
2. 等待合同预览比对弹窗打开，左右两侧文件预览均加载完毕
3. 把右侧差异列表中的 **所有** 内容通过原 channel 返回给用户
4. 等待 **3 秒**后关闭弹窗

### Step 4: 等待工作流完成，人工审核通过

继续使用浏览器工具，在案件详情页等待剩余阶段完成。

**等待机制**：

只需要在【运行流程】Tab 下观察工作流执行进度，**不要**尝试使用其它 Tab 页观察流程

5 秒间隔轮询：

1. 每 5 秒做一次：Bash `sleep 5` → `browser_snapshot`（不要 navigate！）
2. **如何判断 audit_decide 已就绪**：在 snapshot 返回的页面内容中查找 "审核决策"、"建议驳回"、"建议通过"、"建议挂起"、"待审核" 中任一关键词。出现任一关键词立即停止轮询。最多轮询 20 次（约 100 秒）。
3. 查看「审核决策」中给出的反馈，并在相同 channel 中通知用户，等待用户决策
4. 根据用户给出的反馈，执行审核决策操作：
   - 用已有的 browser session 直接 `browser_navigate` 到案件详情页 `${BASE_URL}/contract-audit/cases/${case_no}`
   - 在页面的「审核决策」区域中，找到并点击对应的按钮（【通过】、【驳回】、【挂起】）
   - 在弹出的确认对话框中，填写用户反馈的内容作为审核意见
   - 点击确认按钮提交决策
5. **决策提交后的判停规则（避免无限轮询）**：

   **如果用户决策是【驳回】或【挂起】**：
   - **不需要等待 `system_interact`**，提交决策后立即结束流程，进入第 6 步通知用户。

   **如果用户决策是【通过】**：
   - 以 5 秒间隔轮询（`browser_snapshot`），等待 `system_interact` 完成。
   - 在 snapshot 返回内容中查找以下任一关键词，出现即代表 `system_interact` 完成，**立即停止轮询**：
     - "OA 审批已推送"、"OA审批已推送"
     - "已推送至 OA"、"已推送至OA"
     - "系统交互完成"、"system_interact" 阶段卡片显示"完成"或绿色状态
   - **最多轮询 12 次（约 60 秒）**，超时后直接进入第 6 步，不要继续等待。

6. 处理完毕后，在相同 channel 中通知用户处理的最终结果

### Step 5: 发送消息通知审核结果

在相同 channel 中发送消息，内容为审核通过结论及合同比对要点，格式示例：

```
合同审核通过通知：
审核结论：{审核决策结论}
比对要点：
- 付款条款已修正为净30天
- 补充了违约责任条款
- ...（列出实际比对差异摘要）
审核要点：
- ...（总结审核理由摘要）
```

**注意**：消息内容需保持换行格式，确保每行独立显示，不要合并成一行发送。反馈的消息内容要注意简洁。

### Step 6: 推送到 OA 系统

如果复审结果是【通过】，需要进一步操作 OA 系统。继续复用已有的 browser session，**不要重新调用 `start_browser_session`**。

1. 在浏览器中打开案件详情页 `${BASE_URL}/contract-audit/cases/${case_no}`，在中间最下方的「系统交互」卡片中，点击「OA审批系统」的【查看详情】链接
2. 页面会在浏览器新的 Tab 页中打开「OA 合同审批系统」，在 OA 系统页面中找到【审批流程】 Tab 栏，点击进入审批流程
3. 页面最下方找到审批操作区域
4. 根据审核结果（通过）填写审批意见
5. 点击「同意」按钮完成 OA 审批
