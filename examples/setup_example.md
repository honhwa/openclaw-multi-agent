# 完整工作流示范（主 Agent 视角，方式A）

> 本文件展示你（主 Agent）在真实任务中的完整思考和执行过程。
> 核心原则：**主 Agent 随时保持响应，通过异步模式委派任务，不因等待子 Agent 而卡住。**

---

## 场景：用户请求"帮我实现用户注册功能"

### Step 1：Intent Gate——分析任务

收到："帮我实现用户注册功能"

**我的分析：**
- "实现"注册功能 → 需要规划 + 开发 + 审查，是有依赖的串行复合任务
- 串行顺序：**芒格（规划）→ 费曼（开发）→ 德明（审查）**
- 芒格的 PRD 是费曼的输入，所以芒格必须先完成（用同步）
- 费曼的代码是德明的输入，所以费曼必须先完成（用同步）
- 整体是串行，但每步完成后立即告知用户进展

**先确认：**
- 读取 `~/.openclaw/openclaw.json`，确认 agents.list 中有 munger、feynman、deming ✅
- 确认 `tools.agentToAgent.enabled: true` ✅

**加载当前 Wisdom：**
- 读取 `~/.openclaw/workspace/memory/wisdom/` 下各文件
- 当前 Wisdom 为空（新项目），无需注入

**立即告知用户：**
```
我来帮你实现用户注册功能。这需要三个步骤：
1. 芒格做需求规划和 PRD（进行中...）
2. 费曼根据 PRD 实现代码
3. 德明做代码审查

我会在每步完成后告诉你进展，你可以继续聊其他的。
```

---

### Step 2：发消息给芒格——战略规划（同步，因为费曼需要等这个结果）

```
sessions_send(
  sessionKey="agent:munger:main",
  message="设计用户注册功能的 PRD。

需要包含：
1. 功能描述（用户注册的完整流程）
2. 技术要求（安全性、性能、边界条件）
3. 验收标准（如何判断功能完成）
4. 潜在风险和应对策略

请先用逆向思考列出这个功能可能的失败点，再给出解决方案。
技术栈：Node.js + Express + MongoDB。",
  timeoutSeconds=120  // 串行依赖：费曼需要这个结果才能开始
)
```

**如果用户在等待期间发来新消息（如"另外帮我查一下 XX"）：**
→ 优先回应用户，处理完后再等芒格的结果。

**芒格回复后：**
- 告知用户："✅ 芒格完成了 PRD，开始让费曼实现代码..."
- 提取 Learnings → 写入 Wisdom

**提取 Learnings → 写入 Wisdom：**

新增到 `~/.openclaw/workspace/memory/wisdom/conventions.md`：
```
- **【2026-03-17】芒格：** 密码必须服务端 bcrypt 哈希（rounds=10），禁止明文存储
- **【2026-03-17】芒格：** 邮箱唯一性校验必须在服务端做，前端校验不算
- **【2026-03-17】芒格：** 统一错误响应格式：{ error: { code, message } }
```

---

### Step 3：发消息给费曼——实现代码（同步，因为德明需要等这个结果）

注入 Wisdom + PRD 结果：

```
sessions_send(
  sessionKey="agent:feynman:main",
  message="实现用户注册功能。

【PRD 要求】（来自芒格）：
- 密码要求：最少8位，含大小写+数字
- 邮箱：服务端唯一性校验（MongoDB 唯一索引）
- 限流：每IP每小时最多10次
- 错误响应格式：{ error: { code, message } }

【重要约定】（来自历史经验）：
- 密码必须服务端 bcrypt 哈希（rounds=10），禁止明文存储
- 邮箱唯一性校验必须在服务端做
- MongoDB 11000 错误码需单独处理，返回友好的'邮箱已注册'提示

技术栈：Node.js + Express + MongoDB。
期望输出：完整的 register route 代码，包含输入验证、错误处理、注释。",
  timeoutSeconds=180  // 串行依赖：德明需要这个结果才能开始
)
```

**费曼回复后：**
- 告知用户："✅ 费曼完成了代码，开始让德明审查..."
- 提取 Learnings → 写入 Wisdom

**提取 Learnings → 写入 Wisdom：**

新增到 `~/.openclaw/workspace/memory/wisdom/gotchas.md`：
```
- **【2026-03-17】费曼：** express-rate-limit 中间件必须在路由定义之前注册，否则限流不生效
- **【2026-03-17】费曼：** bcrypt.hash 是异步的，必须用 await，不能用同步版本（性能问题）
```

---

### Step 4：发消息给德明——审查代码（异步，审查结果不需要立即串行到下一步）

注入 Wisdom + 费曼的代码：

```
sessions_send(
  sessionKey="agent:deming:main",
  message="审查以下用户注册功能代码。

【代码】：
[费曼回复的完整代码]

【审查重点】（基于约定和历史教训）：
- ✅ 密码是否用 bcrypt 哈希（不是 MD5 或明文）？
- ✅ 邮箱唯一性是否在服务端校验？
- ✅ 是否处理了 MongoDB 11000 唯一索引错误？
- ✅ 限流中间件是否在路由前注册？
- ✅ 错误响应格式是否为 { error: { code, message } }？
- ✅ 是否处理了空输入、null、undefined？
- ✅ bcrypt.hash 是否用了 await（异步版本）？

请给出清单式审查报告，每项标注 ✅/❌/⚠️，最终给出验收结论。",
  timeoutSeconds=0    // 异步：审查后 announce 结果，主 Agent 保持响应
)
```

**立即告知用户：**
```
德明正在审查代码，完成后会自动通知你结果。
你现在有其他问题要问吗？
```

**德明 announce 结果后：**

```
审查结果：
✅ bcrypt 哈希：await bcrypt.hash(password, 10) ✓
✅ 邮箱唯一性：服务端校验，有唯一索引 ✓
✅ 11000错误处理：返回"邮箱已注册" ✓
✅ 限流中间件：在路由前注册 ✓
✅ 错误格式：{ error: { code, message } } ✓
⚠️ 空输入处理：未处理 password 为空字符串（'' 不同于 undefined）

结论：有条件通过。需修复 ⚠️ 后上线。
```

**提取 Learnings → 写入 Wisdom：**

新增到 `~/.openclaw/workspace/memory/wisdom/failures.md`：
```
- **【2026-03-17】德明：** 费曼未处理 password 空字符串输入。
  教训：输入验证必须显式检查 `password.trim() === ''`，不能只检查 null/undefined。
```

---

### Step 5：处理 ⚠️ 问题——异步修复

```
sessions_send(
  sessionKey="agent:feynman:main",
  message="需要修复注册代码中的一个问题：
未处理 password 为空字符串的情况。

当前的输入验证只检查了 !password（覆盖 undefined/null），
需要同时检查 password.trim() === ''。

请只修改输入验证这一处，返回修改后的代码片段。",
  timeoutSeconds=0    // 异步：小修复，announce 结果即可
)
```

**立即告知用户：**
```
已让费曼修复空字符串验证问题，完成后会通知你。
```

---

### Step 6：汇总回复用户

费曼 announce 修复结果后：

```
✅ 用户注册功能已完成

📋 完成情况：
- PRD 设计：✅ 芒格完成（含逆向分析和验收标准）
- 代码实现：✅ 费曼完成
- 质量审查：✅ 德明审查通过（修复了1个⚠️问题）

📌 本次新增团队约定（已记录到 Wisdom）：
- 密码必须 bcrypt 哈希（rounds=10），禁止明文
- 邮箱唯一性服务端校验，捕获 MongoDB 11000 错误
- 输入验证必须显式检查空字符串
- express-rate-limit 必须在路由前注册

下一步：代码已就绪，可以进行集成测试。
```

---

## 关键要点

1. **串行依赖用同步，独立任务用异步**：芒格→费曼→德明有依赖，前两步用同步；德明审查和费曼修复没有后续依赖，用异步
2. **每步完成后立即告知用户**：不要沉默等待，让用户知道进展
3. **Wisdom 是累积的**：每次任务后提取，下次自动注入，防止重犯错误
4. **用户有新消息时优先回应**：即使在同步等待期间，也要先回应用户的新消息
5. **审查结果驱动后续**：德明发现 ⚠️ → 异步发给费曼修复，形成闭环

---

**版本：** 5.0.0  
**最后更新：** 2026-03-17
