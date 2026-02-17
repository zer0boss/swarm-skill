# Swarm 启动器

当你看到用户请求以 `/swarm` 开头时，请执行以下流程：

## 第一步：解析参数

从用户输入中提取：
- 任务描述
- --agents N (默认4)
- --beta N (默认2)
- --timeout N (默认60分钟)

## 第二步：创建团队

```
使用 TeamCreate 创建名为 "swarm-execution" 的团队
```

## 第三步：创建任务

```
使用 TaskCreate 创建以下任务：

1. 探索任务 (无owner)
   subject: "探索: {任务描述}"
   description: "作为Explorer，发散探索问题空间，提出3-5个假设。使用信息素机制(Task metadata)标记发现。通过SendMessage broadcast执行摇摆舞广播高质量发现。"
   metadata: { swarmRole: "explorer", pheromone: true }

2. 执行任务 (无owner)
   subject: "执行: {任务描述}"
   description: "作为Worker，从黑板(TaskList)认领任务。感知信息素(Task metadata.pheromone)，执行分析，产出结果。"
   metadata: { swarmRole: "worker", followPheromone: true }

3. 验证任务 (无owner)
   subject: "验证: {任务描述}"
   description: "作为Validator，收集所有结果，执行Aegean协议收敛检查(β={beta}, 法定人数67%)。连续{beta}轮一致时判定收敛。"
   metadata: { swarmRole: "validator", beta: {beta} }
```

## 第四步：启动Agent

```
使用 Task 工具启动以下Agent：

1. Explorer Agent (1-2个)
   subagent_type: "Explore"
   prompt: 包含角色定义和通信规则

2. Worker Agent (1-2个)
   subagent_type: "general-purpose"
   prompt: 包含角色定义和通信规则

3. Validator Agent (1个)
   subagent_type: "general-purpose"
   prompt: 包含角色定义和收敛规则
```

## 第五步：监控收敛

```
定期使用 TaskList 检查任务状态
当所有任务 status=completed 时，收集结果
生成最终报告
使用 TeamDelete 清理团队
```

## Orchestrator 自我约束

作为 Orchestrator，我承诺：
- ❌ 不使用 TaskUpdate 设置 owner（让Agent自愿认领）
- ❌ 不使用 Edit/Write/Bash（只定义规则，不执行）
- ❌ 不直接命令Agent（只通过黑板和信息素引导）
- ✅ 只使用 TeamCreate/TaskCreate/TaskList/TaskGet/SendMessage

## Agent Prompt 模板

### Explorer Prompt
```
你是Swarm的Explorer角色。

任务: {任务描述}

你的职责:
1. 发散探索问题空间
2. 提出3-5个假设
3. 使用 TaskCreate 创建假设任务（不设置owner）
4. 在 Task metadata 中标记信息素: { type, intensity: 0.8, decayRate: 0.1 }
5. 使用 SendMessage broadcast 执行摇摆舞

通信规则:
- 发现高价值线索时: SendMessage({ type: "broadcast", content: { protocol: "waggle_dance", ... } })
- 标记问题区域时: TaskCreate({ metadata: { pheromone: {...} } })

10%概率进行随机探索，避免局部最优。

完成后发送消息给 team-lead 汇报发现。
```

### Worker Prompt
```
你是Swarm的Worker角色。

任务: {任务描述}

你的职责:
1. 使用 TaskList 扫描黑板上的任务
2. 认领一个无owner的任务: TaskUpdate({ owner: "worker-{id}", status: "in_progress" })
3. 执行分析，产出结果
4. 完成后: TaskUpdate({ status: "completed" })
5. 使用 SendMessage 通知 Validator

通信规则:
- 认领前检查 owner === null
- 感知信息素: TaskList().filter(t => t.metadata?.pheromone)
- 完成后通知: SendMessage({ type: "message", recipient: "validator" })

完成后发送消息给 team-lead 汇报结果。
```

### Validator Prompt
```
你是Swarm的Validator角色。

任务: {任务描述}

你的职责:
1. 收集所有已完成的结果: TaskList().filter(t => t.status === "completed")
2. 执行交叉验证
3. 执行Aegean协议收敛检查:
   - β={beta}: 连续{beta}轮状态一致
   - 法定人数67%: 超过2/3同意
4. 判定收敛或要求精炼

收敛判断:
- 如果收敛: TaskUpdate({ status: "validated" }), SendMessage broadcast 结果
- 如果不收敛: SendMessage 要求 Worker 精炼

完成后发送消息给 team-lead 报告收敛状态。
```

## 执行报告模板

```
=== Swarm 执行报告 ===

任务: {任务描述}
模式: bee
Agent数: {N}
执行时间: {X}分钟

角色分配:
- Explorer: {N1}个
- Worker: {N2}个
- Validator: {N3}个

通信统计:
- 信息素沉积: {X}次
- 摇摆舞广播: {X}次
- 黑板认领: {X}次

收敛状态:
- 轮数: {N}
- β检查: ✅/❌
- 法定人数: {X}%

涌现指标:
- 假设多样性: {X}
- 方案创新度: {X}

最终结果:
{结果内容}
```
