# AI Native Pipeline 踩坑实录

> 做 Agent 开发，最怕的不是 Agent 不聪明，而是你不知道它什么时候变蠢了。

## 坑一：Agent 改了，不知道效果变好还是变坏

### 问题

改了一个 Agent 的 prompt，感觉效果更好了。但跑了几次发现：

> "之前这个 case 能过的，怎么现在过不了了？"

没有对比，没有量化，全凭感觉。

### 解决：建立评估框架

```
1. 准备测试集（20-50 个典型任务）
2. 运行 Agent，记录输出
3. 对比金标准
4. 生成报告
```

核心指标：

| 指标 | 计算方式 | 目标 |
|------|----------|------|
| 准确率 | 正确输出 / 总输出 | > 80% |
| 完整率 | 覆盖要求 / 总要求 | > 90% |
| 平均耗时 | 总时间 / 任务数 | < 60s |
| Token 消耗 | 总 Token / 任务数 | < 10K |

### 评估方法

**方法 1：金标准对比**

```python
test_cases = [
    {
        "input": "在用户模块增加手机号登录",
        "expected_output": {
            "core_files": ["src/auth/session.py"],
            "risks": ["authenticate() 被多处调用"]
        }
    }
]

def evaluate_agent(agent, test_cases):
    results = []
    for case in test_cases:
        output = agent.run(case["input"])
        score = compare_output(output, case["expected_output"])
        results.append(score)
    return sum(results) / len(results)
```

**方法 2：LLM-as-Judge**

用另一个 LLM 评估输出质量：

```
任务：{input}
Agent 输出：{output}

请打分（1-5分）：
1. 完整性：是否覆盖所有要求？
2. 准确性：信息是否正确？
```

---

## 坑二：Agent 之间信息传递不清晰

### 问题

coding-agent 收到 spec.md，但不知道：
- impact-analyzer 发现了哪些风险
- prd-agent 为什么这样划分优先级
- spec-agent 为什么选择这个 API 设计

**每个 Agent 都是"信息孤岛"**。

### 解决：Task Spec 作为单一事实源

```markdown
# Task Spec

## 上下文（来自 impact-analyzer）
- Core Files: src/auth/session.py
- Risks: authenticate() 被 12 处调用

## 需求（来自 prd-agent）
- 功能列表: ...
- 验收标准: ...

## 技术决策（来自 spec-agent）
- API 设计: POST /api/auth/login
- 为什么这样设计: 项目已有 JWT 工具类

## 实现记录（来自 coding-agent）
- 修改文件: ...
- 关键改动: ...
```

**每个 Agent 都往 Task Spec 里写内容**，后续 Agent 就能看到完整上下文。

---

## 坑三：没有 Planning Gate，错误累积

### 问题

早期版本没有 Planning Gate，Agent 一路跑到底。结果：

- 需求理解错了 → Spec 设计错了 → 代码写错了 → 全白干
- 错误在最后才发现，返工成本高

### 解决：关键节点人工确认

```
[Step 2/5] 分析需求...
  ✅ P0: JWT Token 生成/验证
  ✅ P1: Token 刷新
  
  [PAUSE] 确认需求? [y/n]
```

借鉴 OpenAI 的 **Harness Engineering** 实践：
- Planning Gate 在关键节点暂停
- 用户确认后才继续
- 避免错误假设累积

**两个 Planning Gate**：
- Step 2 后：确认需求范围
- Step 4 后：确认代码改动

---

## 坑四：Token 限制导致上下文丢失

### 问题

大项目的 Task Spec 越来越长，最终超出 token 限制。后面的 Agent 看不到前面的内容。

### 解决：增量上下文 + 智能压缩

**策略 1：只传必要信息**

```python
def get_relevant_context(task_spec: str, agent_role: str) -> str:
    if agent_role == "coding-agent":
        return extract_sections(task_spec, ["技术决策", "Core Files", "验收标准"])
    elif agent_role == "verification-agent":
        return extract_sections(task_spec, ["验收标准", "实现记录"])
```

**策略 2：智能压缩**

```python
def compress_spec(spec: str, max_tokens: int = 8000) -> str:
    # 用 LLM 压缩到目标 token 数
    return llm.invoke(f"压缩以下内容到 {max_tokens} tokens: {spec}")
```

**策略 3：分层存储**

```
Task Spec/
├── summary.md          # 摘要（< 1000 tokens）
├── context.json        # 结构化上下文
└── decisions/          # 决策记录
```

---

## 坑五：中间步骤失败无法恢复

### 问题

Pipeline 执行到一半失败：

```
[Step 1/5] ✅ 代码影响分析
[Step 2/5] ✅ 需求分析
[Step 3/5] ❌ 技术规格设计 - 失败

所有进度丢失，需要从头开始
```

### 解决：断点续传 + 状态持久化

```python
class PipelineState:
    def save_step(self, step: int, result: dict):
        """保存步骤结果"""
        self.state[f"step_{step}"] = {
            "status": "completed",
            "result": result,
            "timestamp": datetime.now().isoformat()
        }
    
    def get_resume_point(self) -> int:
        """获取恢复点"""
        for i in range(1, 6):
            if self.state.get(f"step_{i}", {}).get("status") != "completed":
                return i
        return 6

def run_pipeline(task: str, resume: bool = True):
    state = PipelineState(task_id)
    start_step = state.get_resume_point() if resume else 1
    
    for step in range(start_step, 6):
        try:
            result = run_step(step, task, state)
            state.save_step(step, result)
        except Exception as e:
            print(f"修复后运行: /pipeline --resume {task_id}")
            raise
```

---

## 坑六：复杂任务不知道怎么拆

### 问题

一个任务涉及 5 个 API、3 个模块，直接扔给 coding-agent 会：

- 输出太长，超出 token 限制
- 一个地方出错，整体失败
- 没法并行，效率低

### 解决：自动任务拆解

当 SPEC 复杂度高时（API > 3 或跨模块），自动触发 task-breakdown：

```
[Step 3.5] 任务拆解

| ID | 任务 | 依赖 | 执行 |
|----|------|------|------|
| T1 | 后端 - 登录 API | - | 并行 |
| T2 | 后端 - 验证码 API | - | 并行 |
| T3 | 前端 - 登录页 | - | 并行 |
| T4 | 集成测试 | T1,T2,T3 | 最后 |

执行计划:
- 阶段1（并行）: T1, T2, T3
- 阶段2（验收）: T4
```

---

## 坑七：Agent 不知道何时停止

### 问题

Agent 有时候会：

- 过度优化代码
- 添加不必要的功能
- 无限循环修改

### 解决：明确完成条件

**策略 1：DoD (Definition of Done)**

```python
DEFINITION_OF_DONE = {
    "coding-agent": [
        "所有 P0 需求已实现",
        "测试覆盖率 >= 80%",
        "Lint 检查通过",
    ]
}

def check_completion(agent: str, result: dict) -> bool:
    for criteria in DEFINITION_OF_DONE[agent]:
        if not evaluate_criteria(criteria, result):
            return False
    return True
```

**策略 2：最大迭代限制 + 进度检测**

```python
def run_with_limit(agent, task: str, max_iterations: int = 5):
    previous_result = None
    no_progress_count = 0
    
    for i in range(max_iterations):
        result = agent.run(task)
        
        if check_completion(agent.role, result):
            return result
        
        # 检测是否有实质性进展
        if previous_result and detect_progress(previous_result, result) < 0.1:
            no_progress_count += 1
            if no_progress_count >= 2:
                print("[Warning] 无实质性进展，停止迭代")
                return result
        
        previous_result = result
    
    raise MaxIterationError(f"达到最大迭代次数 {max_iterations}")
```

---

## 坑八：验收标准需要人工预定义

### 问题

传统做法：用户写验收标准

```yaml
verifications:
  - type: test
    command: pytest tests/ -v
```

问题：用户可能不知道怎么写，或者写得不够完整。

### 解决：从 Spec 自动推断

verification-agent 分析 spec.md，自动推断验证项：

```
spec 定义: POST /api/auth/login
→ 自动添加验证: curl 测试登录接口

spec 定义: 数据模型 User
→ 自动添加验证: 检查数据库 schema

spec 定义: 使用 pytest
→ 自动添加验证: pytest tests/ -v
```

这样用户不需要懂测试，Agent 自动生成验收标准。

---

## 坑九：用户反馈无法持续改进

### 问题

用户给反馈后：
- 反馈只对当前任务有效
- 相似任务不会自动应用反馈
- 无法积累改进经验

### 解决：反馈闭环系统

```python
class FeedbackKnowledgeBase:
    def learn(self, task: str, feedback: str, improvement: str):
        """学习反馈"""
        pattern = extract_task_pattern(task)
        self.knowledge[pattern].append({
            "feedback": feedback,
            "improvement": improvement
        })
    
    def get_improvements(self, task: str) -> list[str]:
        """获取相关改进方案"""
        pattern = extract_task_pattern(task)
        return [e["improvement"] for e in self.knowledge.get(pattern, [])]

# 使用
kb = FeedbackKnowledgeBase()

# 学习
kb.learn("读取配置文件", "路径错误", "先用 os.path.exists 检查")

# 应用
improvements = kb.get_improvements("读取配置文件")
# → ["先用 os.path.exists 检查"]
```

---

## 坑十：并发修改冲突

### 问题

多个 Agent 并行执行，同时修改同一个文件：

```
Agent A: 修改 src/auth/login.py
Agent B: 同时修改 src/auth/login.py

结果：Agent B 覆盖了 Agent A 的修改
```

### 解决：执行计划优化

```python
def plan_parallel_execution(tasks: list[Task]) -> ExecutionPlan:
    """规划并行执行，避免文件冲突"""
    
    # 1. 分析每个任务的文件依赖
    file_deps = {t.id: get_modified_files(t) for t in tasks}
    
    # 2. 分组：无冲突的并行，有冲突的串行
    groups = []
    current_group = []
    used_files = set()
    
    for task in tasks:
        task_files = file_deps[task.id]
        
        if task_files & used_files:  # 有冲突
            groups.append(current_group)
            current_group = [task]
            used_files = task_files
        else:  # 无冲突
            current_group.append(task)
            used_files |= task_files
    
    groups.append(current_group)
    
    return ExecutionPlan(parallel_groups=groups)
```

---

## 坑十一：成本失控

### 问题

Pipeline 运行成本：
- LLM 调用费用失控
- Token 消耗无上限
- 没有成本预警

### 解决：成本控制机制

```python
class CostBudget:
    # Token 成本（美元/1K tokens）
    COSTS = {
        "gpt-4": {"input": 0.03, "output": 0.06},
        "gpt-3.5-turbo": {"input": 0.0015, "output": 0.002},
    }
    
    def __init__(self, daily_budget: float = 10.0):
        self.daily_budget = daily_budget
        self.daily_spent = 0.0
    
    def estimate_cost(self, model: str, tokens: int) -> float:
        return (tokens / 1000) * self.COSTS[model]["input"]
    
    def can_proceed(self, estimated_cost: float) -> bool:
        return (self.daily_spent + estimated_cost) <= self.daily_budget
    
    def record(self, cost: float):
        self.daily_spent += cost
        if self.daily_spent > self.daily_budget * 0.8:
            warn(f"成本已使用 {self.daily_spent/self.daily_budget*100:.1f}%")

# 使用
budget = CostBudget(daily_budget=10.0)

if not budget.can_proceed(estimated_cost):
    # 切换到更便宜的模型
    model = "gpt-3.5-turbo"
```

---

## 坑十二：代码分析依赖外部工具

### 问题

impact-analyzer 需要分析代码依赖关系。一开始用 ctags：

```bash
ctags -R --fields=+n --languages=Python ./src/
```

结果：
- 某些机器没装 ctags
- 不同语言需要不同工具

### 解决：无工具降级方案

```python
def analyze_dependencies(project_root: str) -> dict:
    # 尝试用 ctags（精确模式）
    if command_exists("ctags"):
        return analyze_with_ctags(project_root)
    
    # 降级：用 grep + LLM 理解
    imports = run(f"grep -rn 'import\\|from' {project_root}")
    return llm.invoke(f"分析以下代码依赖: {imports}")
```

---

## 总结

做 Agent 开发，核心不是让 Agent 更聪明，而是：

1. **可评估** - 改了 Agent，能知道效果变好还是变坏
2. **可追溯** - 每个 Agent 的决策都有记录
3. **可干预** - 关键节点能暂停，人工确认
4. **可恢复** - 失败后能断点续传
5. **可控** - 知道何时停止，成本可控

**Less is More**：五个节点，每个打磨到极致，比十个节点各做一半要好。

---

## 附录：Check List

```markdown
## 启动前
- [ ] 环境变量配置完整
- [ ] 依赖版本锁定
- [ ] 成本预算设置

## 执行中
- [ ] Token 消耗监控
- [ ] 中间结果保存
- [ ] Planning Gate 确认

## 完成后
- [ ] 质量门禁通过
- [ ] 测试覆盖达标
- [ ] 用户反馈收集
```

---

**项目地址**: https://github.com/afine907/ai-native-pipeline

**上一篇**: [AI Native Pipeline 设计实践](#/articles/ai/AI-Native-Pipeline-设计实践)

**下一篇**: [Agent 效果评估实战](#/articles/ai/Agent-效果评估实战)
