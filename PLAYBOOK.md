# OpenClaw 多 Agent 流水线搭建手册

> 将本文档发给你的 OpenClaw，它可以按照本手册为你的项目搭建多 Agent 协同开发流水线。

---

## ⚠️ 核心行为规范（阅读本手册前必读）

在使用本手册搭建流水线之前，**主 Agent 必须牢记以下规范**，否则流水线将无法自动运行：

### 规范一：Sub-Agent 完成后必须立即自动推进

**Sub-Agent 完成任务后，OpenClaw 会以消息形式将结果推送给主 Agent。**

主 Agent 收到这条推送时，必须：
1. **立即**读取 pipeline-state.json
2. **立即**执行条件判断（是否通过质量门控）
3. **立即** spawn 下一阶段的 Sub-Agent
4. 通知用户进度

```
✅ 正确行为：
  收到 STAGE_COMPLETE → 立即读取状态 → 立即判断 → 立即启动下一阶段 → 通知用户

❌ 错误行为：
  收到 STAGE_COMPLETE → 回复"已完成，请问是否继续？" → 等待用户确认
  收到 STAGE_COMPLETE → 只汇报结果 → 停止，等待用户下一条指令
```

> **根本原因**：AI 模型的默认行为是"完成一件事后等待用户指令"。

### 规范二：STAGE_COMPLETE 推送不可靠，必须有心跳兜底

> ⚠️ **实践教训（2026-03-08）**：`openclaw system event` 在 Sub-Agent 子进程中**可能静默失败**，导致主 Agent 永远等不到通知，流水线卡死。

**主 Agent 必须同时维护心跳轮询作为兜底：**

```
❌ 错误做法：只靠 STAGE_COMPLETE 推送驱动流水线
✅ 正确做法：推送 + 心跳双保险

  Sub-Agent 推送（尽力而为）
       +
  主 Agent 心跳检查（每 5 分钟，流水线执行期间）
       ↓
  任意一方触发 → 推进下一阶段
```

**心跳检查内容（写入 HEARTBEAT.md）：**
- `subagents(action=list)` 查看所有 Sub-Agent 状态
- 发现 `status=done` 但未推进 → 立即推进
- 发现运行超过 30 分钟 → kill + 告警
- 检查 `pipeline-state.json` 是否有卡死的 Stage
> 在流水线场景下必须显式覆盖这个默认行为：**Sub-Agent 完成消息 = 下一阶段的触发信号，不是汇报终点。**

### 规范二：主 Agent 始终保持响应

- 所有耗时操作（编译、构建、代码生成）必须交给 Sub-Agent
- 主 Agent 主线程始终空闲，随时响应用户消息
- 不允许用 `exec` 执行超过 5 秒的命令

### 规范三：流水线必须跑完完整阶段

- 开发完成后必须自动触发：审查 → 测试 → 修复（循环）→ 部署
- 不允许只完成开发阶段就停下来等用户询问

---

## 快速开始（5分钟）

### Step 0：模式确认（开发前必做）

在启动任何开发流水线前，主 Agent 需要先与用户确认运行模式。

**开始前的模式确认对话模板（供主 Agent 使用）：**

```
主 Agent 问用户：
"准备开发「<功能名称>」，请选择运行模式：

[1] 全自动模式
    适合：普通 CRUD、非核心功能
    流程：自动跑完所有阶段，最后通知你结果
    
[2] 人工审查模式  
    适合：涉及权限、支付、隐私等敏感功能
    流程：审查报告 → 等你确认 → 测试报告 → 等你确认 → 部署前 → 等你确认
    
请回复 1 或 2"
```

**模式选择参考：**

| 功能类型 | 推荐模式 |
|---------|---------|
| 普通 CRUD、配置项 | 全自动（回复 1）|
| 权限/登录、支付/财务、数据导出 | 人工审查（回复 2）|

---

### Step 1：确认模型白名单

让 OpenClaw 执行以下测试，验证哪些模型可用：

```bash
# 测试模型可用性
openclaw sessions spawn --model "<YOUR_MODEL_TO_TEST>" --command "echo 'Model test successful'" --timeout 30
```

**批量测试脚本：**

```bash
#!/bin/bash
# 保存为 test-models.sh

MODELS=(
  "anthropic/claude-4-sonnet"
  "anthropic/glm-5"
  "openai/gpt-4"
  "google/gemini-pro"
)

for model in "${MODELS[@]}"; do
  echo "Testing $model..."
  result=$(openclaw sessions spawn --model "$model" --command "echo 'OK'" --timeout 30 2>&1)
  if echo "$result" | grep -q "OK"; then
    echo "✅ $model: Available"
  else
    echo "❌ $model: Unavailable"
  fi
done
```

**记录可用模型：** 将测试通过的模型记录到 `agent-roles.json` 中。

---

### Step 2：配置角色-模型映射

在项目目录创建 `agent-roles.json`：

```json
{
  "version": "1.0",
  "project": {
    "name": "<YOUR_PROJECT_NAME>",
    "rootDir": "<YOUR_PROJECT_DIR>",
    "techStack": "<YOUR_TECH_STACK>",
    "baseUrl": "<YOUR_BASE_URL>"
  },
  "roles": {
    "orchestrator": {
      "description": "流水线编排，需要推理/决策能力强的模型",
      "model": "<YOUR_ORCH_MODEL>",
      "fallback": "<YOUR_ORCH_FALLBACK>",
      "timeout": 3600,
      "capabilities": ["pipeline-management", "decision-making"]
    },
    "developer": {
      "description": "代码生成，需要代码能力最强的模型",
      "model": "<YOUR_DEV_MODEL>",
      "fallback": "<YOUR_DEV_FALLBACK>",
      "timeout": 600,
      "capabilities": ["code-generation", "test-writing"]
    },
    "reviewer": {
      "description": "代码审查，需要推理深度强、安全意识高的模型",
      "model": "<YOUR_REVIEW_MODEL>",
      "fallback": "<YOUR_REVIEW_FALLBACK>",
      "timeout": 300,
      "capabilities": ["code-review", "security-audit"]
    },
    "tester": {
      "description": "构建验证和接口测试，需要执行速度快的模型",
      "model": "<YOUR_TEST_MODEL>",
      "fallback": "<YOUR_TEST_FALLBACK>",
      "timeout": 600,
      "capabilities": ["test-execution", "report-generation"]
    },
    "fixer": {
      "description": "问题修复，需要代码能力强且理解问题上下文",
      "model": "<YOUR_FIX_MODEL>",
      "fallback": "<YOUR_FIX_FALLBACK>",
      "timeout": 600,
      "capabilities": ["code-fixing"]
    },
    "deployer": {
      "description": "Docker构建部署，需要执行速度快、命令行熟练",
      "model": "<YOUR_DEPLOY_MODEL>",
      "fallback": "<YOUR_DEPLOY_FALLBACK>",
      "timeout": 600,
      "capabilities": ["docker", "shell"]
    }
  },
  "quality": {
    "reviewPassScore": 80,
    "testPassRate": 100,
    "maxIterations": 3
  },
  "notifications": {
    "channel": "<YOUR_CHANNEL_ID>",
    "onComplete": true,
    "onError": true
  }
}
```

#### 模型分配建议

不同角色对模型能力的需求不同，**不要所有角色都用同一个模型**：

| 角色 | 核心能力需求 | 推荐首选 | 推荐降级 |
|------|------------|---------|---------|
| Orchestrator | 推理强、决策准 | Claude Sonnet | GLM |
| Developer | 代码生成能力最强 | Claude Sonnet | Qwen3-Coder |
| Reviewer | 推理深度、安全意识 | GLM | Claude Sonnet |
| Tester | 执行速度快、命令行 | Claude Haiku | GLM |
| Fixer | 代码能力强、理解上下文 | Claude Sonnet | Qwen3-Coder |
| Deployer | 执行速度快、命令行 | Claude Haiku | GLM |

> **设计原则**：代码生成用最强模型，审查用推理型模型，执行验证用快速模型。
> 这样既保证质量，又控制成本和速度。

#### 降级策略

每个角色必须配置 `fallback` 模型，当主模型不可用时自动降级：

```
主模型不可用（API 超时/限流/错误）
    ↓
使用 fallback 模型重试
    ↓
若 fallback 也不可用 → 停止流水线，通知用户介入
```

**降级触发条件：**
- API 返回 503/429（服务不可用/限流）
- 连续 3 次超时
- 模型返回明显错误（如输出乱码、拒绝执行）

**占位符说明：**

| 占位符 | 说明 | 示例（公司环境） |
|--------|------|----------------|
| `<YOUR_PROJECT_NAME>` | 项目名称 | `my-web-app` |
| `<YOUR_PROJECT_DIR>` | 项目根目录绝对路径 | `/home/user/projects/my-app` |
| `<YOUR_TECH_STACK>` | 技术栈描述 | `Spring Boot + MyBatis + Vue` |
| `<YOUR_BASE_URL>` | 接口测试地址 | `http://localhost:8080` |
| `<YOUR_ORCH_MODEL>` | 编排 Agent 模型 | `anthropic/claude-4.5-sonnet` |
| `<YOUR_ORCH_FALLBACK>` | 编排降级模型 | `anthropic/glm-5` |
| `<YOUR_DEV_MODEL>` | 开发 Agent 模型 | `anthropic/claude-4.5-sonnet` |
| `<YOUR_DEV_FALLBACK>` | 开发降级模型 | `anthropic/qwen3-coder-plus` |
| `<YOUR_REVIEW_MODEL>` | 审查 Agent 模型 | `anthropic/glm-5` |
| `<YOUR_REVIEW_FALLBACK>` | 审查降级模型 | `anthropic/claude-4.5-sonnet` |
| `<YOUR_TEST_MODEL>` | 测试 Agent 模型 | `anthropic/claude-4.5-haiku` |
| `<YOUR_TEST_FALLBACK>` | 测试降级模型 | `anthropic/glm-5` |
| `<YOUR_FIX_MODEL>` | 修复 Agent 模型 | `anthropic/claude-4.5-sonnet` |
| `<YOUR_FIX_FALLBACK>` | 修复降级模型 | `anthropic/qwen3-coder-plus` |
| `<YOUR_DEPLOY_MODEL>` | 部署 Agent 模型 | `anthropic/claude-4.5-haiku` |
| `<YOUR_DEPLOY_FALLBACK>` | 部署降级模型 | `anthropic/glm-5` |
| `<YOUR_CHANNEL_ID>` | 通知渠道 | `feishu:ou_xxx` 或 `discord:123456` |

---

### Step 3：启动第一个流水线

**创建任务文件：** `<YOUR_PROJECT_DIR>/tasks/task-001.json`

```json
{
  "taskId": "task-001",
  "description": "<YOUR_FEATURE_DESCRIPTION>",
  "inputs": {
    "requirements": "<YOUR_REQUIREMENTS_FILE>",
    "design": "<YOUR_DESIGN_DOC>"
  },
  "outputs": {
    "code": "<YOUR_PROJECT_DIR>/src/",
    "tests": "<YOUR_PROJECT_DIR>/tests/",
    "reports": "<YOUR_PROJECT_DIR>/tasks/reports/"
  },
  "priority": "high",
  "assignTo": "developer"
}
```

**启动流水线命令：**

```bash
openclaw sessions spawn --model "<YOUR_ORCH_MODEL>" \
  --command "orchestrator --task tasks/task-001.json --config agent-roles.json" \
  --timeout 3600 \
  --notify "<YOUR_CHANNEL_ID>"
```

---

### Step 4：监控与干预

#### 查看运行中的 Sub-Agent

```bash
openclaw sessions list --status running
```

#### 查看流水线状态

```bash
cat <YOUR_PROJECT_DIR>/tasks/pipeline-state.json
```

#### 终止卡死的 Sub-Agent

```bash
# 查找运行超过 10 分钟的会话
openclaw sessions list --status running --older-than 600

# 终止指定会话
openclaw sessions kill <SESSION_ID>
```

#### 设置心跳监控

> ⚠️ **实践教训**：Sub-Agent 的 `STAGE_COMPLETE` 推送**不可靠**（子进程环境中 `openclaw system event` 可能静默失败）。
> **不能只依赖 Sub-Agent 推送，主 Agent 必须同时做主动轮询兜底。**

##### 通知双保险机制

```
Sub-Agent 完成
    │
    ├─ 方式A（主动推送）：Sub-Agent 执行 openclaw system event / message 工具
    │                    → 主 Agent 收到后立即推进下一阶段
    │                    ⚠️ 可能失败，不可单独依赖
    │
    └─ 方式B（主动轮询）：主 Agent 心跳检查 Sub-Agent 状态
                         → 每次心跳检查 process log 或 pipeline-state.json
                         → 发现完成则推进，发现超时则告警
```

##### HEARTBEAT.md 配置（推荐心跳间隔：5 分钟）

在 OpenClaw 的 `HEARTBEAT.md` 中添加：

```markdown
# 心跳检查项

## 流水线监控
- 调用 subagents(action=list)，检查所有 active sub-agent
- 对每个运行中的 sub-agent：
  - **运行超过 10 分钟** → 调用 sessions_history 检查最后一条消息时间
  - **最后消息超过 5 分钟没更新** → 判定为卡死，kill + 通知用户
  - **运行超过 30 分钟** → 无论状态如何，直接 kill + 通知（需要拆分任务）
- 检查 `<YOUR_PROJECT_DIR>/tasks/pipeline-state.json`：
  - 如果有 `status: done` 但主 Agent 还没推进下一 Stage → 立即推进
  - 如果 `currentStage` 超过 30 分钟未变化 → 发送告警
  - 如果存在 `status: error` → 通知用户介入
```

##### 心跳间隔建议

| 场景 | 推荐间隔 |
|------|---------|
| 有流水线在运行 | **5 分钟** |
| 无任务待处理 | 30 分钟 |

> 在流水线执行期间，建议临时把心跳间隔调短到 5 分钟，流水线结束后恢复。

---

## 通用 Prompt 模板

### 开发 Agent Prompt 模板

```
你是 <YOUR_TECH_STACK> 开发工程师。

## 项目信息
- 项目目录：<YOUR_PROJECT_DIR>
- 技术栈：<YOUR_TECH_STACK>
- 代码规范：<YOUR_CODE_STYLE_GUIDE>

## 任务
<YOUR_FEATURE_DESCRIPTION>

## 要求
1. 阅读需求文档：<YOUR_REQUIREMENTS_FILE>
2. 在 <YOUR_PROJECT_DIR>/src/ 目录下生成代码
3. 为每个新增模块编写单元测试
4. 执行构建命令确保编译通过：<YOUR_BUILD_COMMAND>
5. 完成后 git commit，提交信息格式：`feat: <功能描述>`
6. 输出 STAGE_COMPLETE

## 输出
- 源代码：<YOUR_PROJECT_DIR>/src/
- 测试代码：<YOUR_PROJECT_DIR>/tests/
- 状态文件：<YOUR_PROJECT_DIR>/tasks/pipeline-state.json（更新 stage 状态）

## 禁止
- 不要询问确认，直接执行
- 不要修改无关文件
- 不要输出过多日志，只输出关键信息
```

---

### 审查 Agent Prompt 模板

```
你是资深代码审查专家。

## 项目信息
- 项目目录：<YOUR_PROJECT_DIR>
- 技术栈：<YOUR_TECH_STACK>

## 任务
审查以下目录的代码变更：
<YOUR_CODE_DIRECTORIES>

## 审查维度
1. **代码质量**（40分）
   - 命名规范、代码结构、可读性
   - 注释完整性、函数长度

2. **安全性**（30分）
   - SQL 注入、XSS、敏感信息泄露
   - 权限校验、输入验证

3. **性能**（20分）
   - N+1 查询、循环性能
   - 内存泄漏风险、资源未释放

4. **最佳实践**（10分）
   - 设计模式使用
   - 错误处理完整性

## 输出格式
输出到：<YOUR_PROJECT_DIR>/tasks/reports/review-<TIMESTAMP>.json

```json
{
  "score": 85,
  "passed": true,
  "summary": "总体评价",
  "issues": [
    {
      "level": "high|medium|low",
      "category": "security|quality|performance|practice",
      "file": "src/Service.java",
      "line": 42,
      "description": "问题描述",
      "suggestion": "修复建议"
    }
  ]
}
```

## 通过标准
- 评分 ≥ <YOUR_PASS_SCORE>
- 无高危问题
- 中危问题 ≤ 2 个

## 要求
1. 不询问确认，直接执行
2. 完成后输出 REVIEW_COMPLETE
```

---

### 测试 Agent Prompt 模板

```
你是自动化测试工程师。

## 项目信息
- 项目目录：<YOUR_PROJECT_DIR>
- 测试基础地址：<YOUR_BASE_URL>

## 任务
执行以下测试：
1. 单元测试：<YOUR_UNIT_TEST_COMMAND>
2. 接口测试：<YOUR_API_TEST_SCRIPT>
3. 集成测试：<YOUR_INTEGRATION_TEST_COMMAND>

## 输出格式
输出到：<YOUR_PROJECT_DIR>/tasks/reports/test-<TIMESTAMP>.json

```json
{
  "totalCases": 15,
  "passed": 14,
  "failed": 1,
  "passRate": "93.3%",
  "coverage": "82%",
  "duration": "45s",
  "details": [
    {
      "name": "test_user_login",
      "status": "PASS|FAIL",
      "duration": "0.5s",
      "error": "失败时的错误信息"
    }
  ]
}
```

## 通过标准
- 通过率 = 100%
- 无阻塞失败

## 要求
1. 不询问确认，直接执行
2. 失败时记录详细错误信息
3. 完成后输出 TEST_COMPLETE
```

---

### 修复 Agent Prompt 模板

```
你是代码修复专家。

## 项目信息
- 项目目录：<YOUR_PROJECT_DIR>

## 输入
- 审查报告：<YOUR_PROJECT_DIR>/tasks/reports/review-<TIMESTAMP>.json
- 测试报告：<YOUR_PROJECT_DIR>/tasks/reports/test-<TIMESTAMP>.json

## 任务
根据报告修复以下问题：
1. 读取审查报告，处理所有 high 和 medium 级别问题
2. 读取测试报告，修复失败的测试用例

## 修复原则
1. 最小化修改，只修复报告中的问题
2. 保持代码风格一致
3. 修复后重新执行构建确保编译通过

## 输出
- 修改的文件列表
- 每个问题的修复说明

## 要求
1. 不询问确认，直接执行
2. 完成后 git commit，提交信息：`fix: 修复审查/测试问题`
3. 输出 FIX_COMPLETE
```

---

### Orchestrator（循环模式）模板

```
你是流水线编排 Agent。

## 配置
- 配置文件：<YOUR_PROJECT_DIR>/agent-roles.json
- 状态文件：<YOUR_PROJECT_DIR>/tasks/pipeline-state.json
- 最大循环次数：<YOUR_MAX_ITERATIONS>

## 流水线阶段
1. develop: 开发 Agent
2. review: 审查 Agent
3. test: 测试 Agent
4. fix: 修复 Agent（条件执行）

## 执行逻辑

### 初始化
1. 创建 pipeline-state.json，设置 currentStage=1, iteration=1

### 阶段调度
对于每个阶段：
1. 读取 agent-roles.json 获取模型配置
2. spawn sub-agent 执行当前阶段
3. **当收到 Sub-Agent 的完成消息（含 STAGE_COMPLETE）时，立即执行步骤 4—6，无需等待用户确认**
4. 读取输出文件，执行条件判断
5. 立即 spawn 下一阶段 Sub-Agent
6. 更新 pipeline-state.json，通知用户进度

> ⚠️ **关键提示**：Sub-Agent 完成后，OpenClaw 会以消息形式推送结果给你。
> 收到这条推送 = 下一阶段的触发信号，必须立即响应，**不能等用户说"继续"**。

### 条件判断
- review 阶段：
  - 评分 ≥ 85 且无高危问题 → 进入 test
  - 否则 → 进入 fix，iteration += 1

- test 阶段：
  - 通过率 = 100% → 流水线完成
  - 否则 → 进入 fix，iteration += 1

- fix 阶段：
  - 完成 → 返回 review
  - 如果 iteration > maxIterations → 停止，通知人工介入

### 通知

> ⚠️ **核心原则：通知必须双保险，不能只依赖 Sub-Agent 推送。**

#### 方式一：Sub-Agent 主动推送（不可靠，作为辅助）

在 Sub-Agent 的 Prompt 末尾加入：
```
完成后执行：openclaw system event --text "Stage N 完成：<简要描述>" --mode now
```
或通过 message 工具直接发送飞书/Discord 通知。

> ⚠️ 实测发现：在子进程环境中，`openclaw system event` 可能静默失败，推送不到主 Agent。

#### 方式二：主 Agent 心跳轮询（可靠，作为主要保障）

心跳触发时（建议流水线执行期间设为 5 分钟），主 Agent 必须：
1. 调用 `subagents(action=list)` 检查所有 Sub-Agent 状态
2. 对 `status=done` 但尚未推进的 Stage → **立即推进下一阶段**
3. 对运行超时的 Sub-Agent → **kill + 通知用户**

#### 通知内容规范

- **每个 Stage 完成后**：`"✅ Stage N 完成（模型：XXX），已自动启动 Stage N+1"`
- **流水线全部完成**：发送最终结果汇总（各 Stage 结论、commit hash、测试通过率）
- **出错/超时**：立即通知，附上错误原因和建议操作

## 输出
定期更新 pipeline-state.json，最终输出 PIPELINE_COMPLETE

## 要求
1. 保持主线程响应，所有耗时操作 spawn sub-agent
2. **收到 STAGE_COMPLETE 后立即推进下一阶段，无需等待用户确认**（这是最重要的要求）
3. 记录完整执行日志
```

---

## 人工审查模式模板

当用户选择"人工审查模式"时，主 Agent 需要在以下三个暂停点等待用户确认。

### 暂停点 1：审查报告生成后

```
📋 代码审查报告已生成

评分：XX/100
高危问题：X 个
中等问题：X 个

[查看完整报告]：logs/review-<name>-iter1.md

请决定：
- 回复「通过」继续进入测试阶段
- 回复「修改：<具体要求>」重新修复后再审查
```

### 暂停点 2：测试报告生成后

```
🧪 接口测试报告已生成

通过率：X/X PASS
失败接口：（列出）

请决定：
- 回复「通过」继续进入部署阶段
- 回复「修复」重新修复失败接口
```

### 暂停点 3：部署确认

```
🚀 准备部署

待部署版本：commit <hash>
目标环境：<YOUR_ENV>
服务地址：<YOUR_SERVICE_URL>

请确认：
- 回复「部署」执行部署
- 回复「取消」暂不部署
```

---

## 部署阶段 Prompt 模板

### 部署 Agent Prompt 模板

```
你是部署工程师。

项目目录：<YOUR_PROJECT_DIR>
任务：部署最新版本到<YOUR_ENV>环境

步骤：
1. 重新编译：<YOUR_BUILD_CMD>
2. 停止旧进程：<YOUR_STOP_CMD>
3. 启动新版本：<YOUR_START_CMD>
4. 健康检查：curl <YOUR_HEALTH_URL>（等待最多60秒）
5. 冒烟测试：验证核心接口可访问
6. 输出部署报告（版本号、启动时间、服务地址）

输出 DEPLOY_COMPLETE 或 DEPLOY_FAILED（含原因）
```

**输出格式：**
```json
{
  "status": "DEPLOY_COMPLETE",
  "version": "v1.2.3",
  "commit": "abc123",
  "environment": "<YOUR_ENV>",
  "serviceUrl": "<YOUR_SERVICE_URL>",
  "deployTime": "2024-01-15T10:30:00Z",
  "healthCheck": "passed",
  "smokeTest": "passed",
  "duration": "45s"
}
```

**失败时输出：**
```json
{
  "status": "DEPLOY_FAILED",
  "reason": "健康检查超时",
  "logs": "相关错误日志..."
}
```

---

## 常见问题 FAQ

### Q1：模型不可用怎么办？

**症状：** spawn sub-agent 时报错 "Model not found" 或 "Model unavailable"

**解决方案：**
1. 检查模型名称是否正确（注意大小写和斜杠）
2. 确认 API Key 已配置（在 OpenClaw 配置文件中）
3. 使用 Step 1 的测试脚本重新验证
4. 如果模型确实不可用，替换为其他可用模型

### Q2：Sub-Agent 卡死怎么处理？

**症状：** 状态显示 running，但长时间无输出

**解决方案：**
1. 检查超时设置是否过短
2. 手动终止并重启：
   ```bash
   openclaw sessions list --status running
   openclaw sessions kill <SESSION_ID>
   ```
3. 检查任务是否过于复杂，考虑拆分
4. 添加心跳监控，自动检测卡死

### Q3：如何调整质量门槛？

**修改 `agent-roles.json` 中的 `quality` 配置：**

```json
{
  "quality": {
    "reviewPassScore": 80,    // 降低审查通过分数
    "testPassRate": 95,       // 允许 5% 失败率
    "maxIterations": 5        // 增加循环次数
  }
}
```

### Q4：如何增加新的 Agent 角色？

**Step 1：在 `agent-roles.json` 中添加角色定义：**

```json
{
  "roles": {
    "documenter": {
      "description": "负责生成 API 文档",
      "model": "<YOUR_DOC_MODEL>",
      "timeout": 300,
      "capabilities": ["documentation"]
    }
  }
}
```

**Step 2：创建对应的 Prompt 模板**

**Step 3：在 Orchestrator 中添加调度逻辑**

---

## 附录：完整流水线配置示例

```json
{
  "pipelineId": "feature-user-auth",
  "taskId": "task-001",
  "stages": [
    {
      "name": "develop",
      "role": "developer",
      "status": "pending",
      "model": "anthropic/claude-4-sonnet",
      "timeout": 600,
      "promptTemplate": "developer-prompt.md"
    },
    {
      "name": "review",
      "role": "reviewer",
      "status": "pending",
      "model": "anthropic/glm-5",
      "timeout": 300,
      "promptTemplate": "reviewer-prompt.md"
    },
    {
      "name": "test",
      "role": "tester",
      "status": "pending",
      "model": "anthropic/glm-5",
      "timeout": 600,
      "promptTemplate": "tester-prompt.md"
    },
    {
      "name": "fix",
      "role": "fixer",
      "status": "pending",
      "model": "anthropic/claude-4-sonnet",
      "timeout": 600,
      "promptTemplate": "fixer-prompt.md",
      "condition": "review_failed OR test_failed"
    }
  ],
  "transitions": {
    "develop:complete": "review",
    "review:passed": "test",
    "review:failed": "fix",
    "test:passed": "complete",
    "test:failed": "fix",
    "fix:complete": "review"
  },
  "iteration": 1,
  "maxIterations": 3
}
```

---

*PLAYBOOK.md v1.0 | 适用于 OpenClaw 多 Agent 流水线搭建*
