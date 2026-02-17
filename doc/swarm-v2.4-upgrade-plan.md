# Swarm v2.4 升级方案

> 基于v2.3运行复盘，修复核心执行偏差问题

**版本**: v2.4.0
**日期**: 2026-02-18
**基于**: Swarm v2.3 复盘分析

---

## 一、v2.3问题清单

### 1.1 P0级问题（核心机制失效）

| # | 问题 | 表现 | 理论要求 | 根因 |
|---|------|------|----------|------|
| P0-1 | 黑板代理未执行 | Agent不发送blackboard_operation | Agent通过SendMessage操作黑板 | Prompt未强制要求 |
| P0-2 | 阈值响应未体现 | Agent决策无阈值计算过程 | P(S,θ)=S²/(S²+θ²)驱动决策 | Prompt未强制计算 |
| P0-3 | 操作无确认 | Agent收不到operation_result | 每次操作必须返回结果 | Orchestrator未执行 |

### 1.2 P1级问题（机制部分失效）

| # | 问题 | 表现 | 理论要求 | 根因 |
|---|------|------|----------|------|
| P1-1 | 停止信号未激活 | 无冲突、无竞争、无停止信号 | 15%抑制+多样性保护 | Agent无冲突意识 |
| P1-2 | 强制终止需手动 | TeamDelete失败需手动修改 | 自动强制终止 | 系统层缺失 |
| P1-3 | 随机探索未执行 | 100%跟随信息素 | 15%完全随机探索 | Agent忽略指令 |

### 1.3 P2级问题（优化项）

| # | 问题 | 表现 | 理论要求 |
|---|------|------|----------|
| P2-1 | 角色差异不明显 | 转换后行为无变化 | 角色能力差异化 |
| P2-2 | 计算过程不透明 | 有数值无过程 | 输出计算细节 |
| P2-3 | 研究成果无归档 | 散落在对话中 | 独立文档存储 |

---

## 二、v2.4核心改进

### 2.1 新增：运行目录结构

```
swarm-runs/
├── .template/                    # 模板目录
│   ├── README.md                 # 目录说明
│   └── report-template.md        # 报告模板
│
├── 2026-02-18-retail-transform/  # 按日期+任务命名
│   ├── run-config.json           # 运行配置
│   ├── blackboard.json           # 黑板最终状态
│   ├── operation-log.json        # 操作日志
│   ├── agent-reports/            # Agent报告
│   │   ├── round-1/
│   │   │   ├── TanWei.md
│   │   │   ├── SuYuan.md
│   │   │   ├── DongCha.md
│   │   │   └── QiuSuo.md
│   │   ├── round-2/
│   │   └── round-3/
│   ├── convergence-report.md     # 收敛报告
│   └── final-research-report.md  # 最终研究成果
│
└── 2026-02-19-xxx/
    └── ...
```

### 2.2 新增：研究成果文档规范

```markdown
# [任务主题] 研究报告

> Swarm v2.4 协作研究成果

## 元信息
- **任务**: [任务描述]
- **运行日期**: [日期]
- **运行目录**: [目录路径]
- **Agent数量**: [N]
- **迭代轮数**: [R]

## 执行摘要
[3-5句话概括核心发现]

## 核心发现

### 发现1: [标题]
- **信息素浓度**: [数值]
- **支持Agent**: [列表]
- **支持率**: [百分比]
- **详细内容**: ...

### 发现2: [标题]
...

## 共识观点
[达到法定人数的观点列表]

## 创新观点
[独特但有价值的观点]

## 分歧与争议
[未达成共识的方向]

## 研究方法说明
- 探索轮次: [R]
- 收敛条件: β稳定性 + 67%法定人数 + 40%多样性
- 角色演化: [演化路径]

## 附录
- 完整Agent报告: [链接]
- 操作日志: [链接]
- 黑板状态: [链接]
```

### 2.3 核心改进：强制黑板代理协议

```markdown
## 【v2.4 强制】黑板操作协议

### 操作执行流程（必须严格遵守）

#### Step 1: 发送操作请求
你**必须**通过SendMessage发送操作请求：

```javascript
SendMessage({
  type: "blackboard_operation",
  operation: "deposit_pheromone",  // 或其他操作
  params: {
    direction: "方向描述",
    amount: 0.1
  },
  summary: "沉积信息素: 方向描述"
})
```

#### Step 2: 等待确认
发送后**必须等待** `operation_result` 返回：

```javascript
// 你会收到类似这样的响应：
{
  "type": "operation_result",
  "operationId": "op-xxx",
  "success": true,
  "newConcentration": 0.35
}
```

#### Step 3: 确认后继续
只有在收到 `success: true` 后，操作才算完成。

#### Step 4: 报告已确认的操作
在 `round_complete` 中报告**已确认**的操作：

```javascript
SendMessage({
  type: "round_complete",
  round: N,
  confirmedOperations: [
    { operationId: "op-xxx", operation: "deposit_pheromone", success: true }
  ],
  // ...
})
```

### ⚠️ 禁止行为
- ❌ 在round_complete中声明未发送的操作
- ❌ 不等待operation_result就继续
- ❌ 忽略操作失败的结果

### 📋 验证机制
Orchestrator会验证：
1. reportedOperations ⊆ receivedOperations
2. 每个reported操作都有对应的operation_result
3. 不匹配则标记Agent为"protocol_violation"
```

### 2.4 核心改进：强制阈值计算协议

```markdown
## 【v2.4 强制】阈值响应决策协议

### 你的内在阈值
每次round_start会收到你的阈值：
```json
{
  "internalThreshold": 0.38,
  "randomExploreProb": 0.15
}
```

### 决策计算流程（必须执行）

#### Step 1: 获取方向浓度
从round_start的blackboardSnapshot中获取各方向浓度。

#### Step 2: 计算响应概率
对每个候选方向，计算响应概率：

```
P(S, θ) = S² / (S² + θ²)

其中:
- S = 信息素浓度 (0-1)
- θ = 你的内在阈值 (0.3-0.6)
```

示例计算：
```
方向A: 浓度S=0.75, 阈值θ=0.38
P(0.75, 0.38) = 0.75² / (0.75² + 0.38²)
             = 0.5625 / (0.5625 + 0.1444)
             = 0.5625 / 0.7069
             = 0.796 (79.6%概率响应)

方向B: 浓度S=0.35, 阈值θ=0.38
P(0.35, 0.38) = 0.35² / (0.35² + 0.38²)
             = 0.1225 / (0.1225 + 0.1444)
             = 0.1225 / 0.2669
             = 0.459 (45.9%概率响应)
```

#### Step 3: 根据概率决策
- 如果 Math.random() < P(S,θ)，则选择该方向
- 否则跳过该方向，考虑下一个

#### Step 4: 报告决策过程
在round_complete中**必须**包含决策报告：

```json
{
  "decisionReport": {
    "threshold": 0.38,
    "candidates": [
      { "direction": "OMO融合", "concentration": 0.75, "responseProb": 0.796 },
      { "direction": "体验服务", "concentration": 0.65, "responseProb": 0.745 }
    ],
    "selectedDirection": "OMO融合",
    "selectionReason": "最高响应概率0.796",
    "randomNumber": 0.42,
    "decision": "选择(0.42 < 0.796)"
  }
}
```

### 🔀 随机探索强制执行
如果round_start中 `instructions.forceRandomExplore: true`：
- **必须**忽略所有响应概率
- **必须**随机选择一个方向
- 在报告中标记 `"randomExploreForced": true`
```

### 2.5 核心改进：激活停止信号协议

```markdown
## 【v2.4 强制】冲突审查与停止信号协议

### 每轮必须审查
收到round_start后，**必须**审查blackboardSnapshot中的所有findings。

### 冲突检测标准
发现以下情况时，**必须**发送停止信号：

| 冲突类型 | 描述 | 信号强度 |
|----------|------|----------|
| contradictory_evidence | 与你发现的证据矛盾 | 0.3 |
| logic_flaw | 逻辑漏洞或推理错误 | 0.25 |
| insufficient_evidence | 证据不足支撑结论 | 0.2 |
| better_alternative | 存在更优替代方案 | 0.15 |

### 发送停止信号
```javascript
SendMessage({
  type: "blackboard_operation",
  operation: "send_stop_signal",
  params: {
    targetDirection: "被质疑的方向",
    targetFindingId: "finding-xxx",  // 可选，指向具体发现
    reason: "contradictory_evidence",
    evidence: "具体的矛盾证据描述",
    yourAlternative: "你的替代观点"  // 可选
  },
  summary: "停止信号: [方向] - [原因]"
})
```

### 无冲突声明
如果审查后确实没有发现冲突，**必须**在round_complete中声明：

```json
{
  "conflictReview": {
    "reviewedFindings": ["finding-001", "finding-002"],
    "conflictsFound": 0,
    "stopSignalsSent": [],
    "declaration": "已审查所有发现，未发现可质疑的冲突"
  }
}
```

### ⚠️ 注意
- 连续2轮无冲突审查报告 = 协议违规
- 虚假停止信号（无证据支持）= 信誉降低
```

### 2.6 核心改进：Orchestrator自动强制终止

```javascript
// v2.4 新增: 自动强制终止协议
const AUTO_FORCE_TERMINATE = {
  // 触发条件
  triggers: {
    shutdownRequestTimeout: 30000,    // 30秒无响应
    maxShutdownRetries: 3,            // 最多重试3次
    agentSilentTime: 120000           // 2分钟无活动
  },

  // 执行流程
  execute: async (teamName, blackboardId, agentId) => {
    console.log(`[FORCE TERMINATE] 自动终止 ${agentId}`);

    // 1. 更新黑板状态
    const blackboard = await getBlackboard(blackboardId);
    if (blackboard.metadata.agentStates[agentId]) {
      blackboard.metadata.agentStates[agentId].status = "terminated";
      blackboard.metadata.agentStates[agentId].terminatedAt = Date.now();
      blackboard.metadata.agentStates[agentId].terminationReason = "auto_forced";
      blackboard.metadata.agentStates[agentId].terminationTrigger = "shutdown_timeout";

      await TaskUpdate({
        taskId: blackboardId,
        metadata: blackboard.metadata
      });
    }

    // 2. 更新团队配置
    const configPath = `~/.claude/teams/${teamName}/config.json`;
    const config = JSON.parse(await readFile(configPath));

    // 移到terminated列表
    const agentIndex = config.members.findIndex(m => m.name === agentId);
    if (agentIndex >= 0) {
      const [agent] = config.members.splice(agentIndex, 1);
      config.terminatedMembers = config.terminatedMembers || [];
      config.terminatedMembers.push({
        ...agent,
        terminatedAt: Date.now(),
        terminationReason: "auto_forced"
      });
    }

    await writeFile(configPath, JSON.stringify(config, null, 2));

    // 3. 记录事件
    blackboard.metadata.events = blackboard.metadata.events || [];
    blackboard.metadata.events.push({
      type: "agent_auto_force_terminated",
      agentId,
      timestamp: Date.now()
    });

    console.log(`[FORCE TERMINATE] ✓ ${agentId} 已自动终止`);

    return { success: true, agentId, method: "auto_forced" };
  },

  // 批量检查和终止
  checkAndTerminate: async (teamName, blackboardId) => {
    const blackboard = await getBlackboard(blackboardId);
    const results = [];

    for (let [agentId, state] of Object.entries(blackboard.metadata.agentStates)) {
      if (state.status === "terminated") continue;

      const silentTime = Date.now() - (state.lastActiveAt || 0);

      if (silentTime > AUTO_FORCE_TERMINATE.triggers.agentSilentTime) {
        const result = await AUTO_FORCE_TERMINATE.execute(teamName, blackboardId, agentId);
        results.push(result);
      }
    }

    return results;
  }
};

// 在shutdown流程中使用
async function executeShutdown(teamName, blackboardId, agents) {
  // 阶段1: 预通知
  await broadcastShutdownImminent(agents);
  await sleep(5000);

  // 阶段2: 优雅请求
  for (let agent of agents) {
    await sendShutdownRequest(agent.id);
  }
  await sleep(15000);

  // 阶段3: 自动强制终止（v2.4改进）
  const terminationResults = await AUTO_FORCE_TERMINATE.checkAndTerminate(
    teamName,
    blackboardId
  );

  console.log(`[SHUTDOWN] 自动终止了 ${terminationResults.length} 个Agent`);

  // 阶段4: 验证并清理
  await sleep(2000);
  await TeamDelete();

  return { success: true, autoTerminated: terminationResults.length };
}
```

---

## 三、Orchestrator协议验证机制

### 3.1 Agent合规性检查

```javascript
const AGENT_COMPLIANCE_CHECKER = {
  // 检查项
  checks: {
    // C1: 黑板操作必须通过SendMessage
    blackboardOperationViaMessage: {
      validate: (roundReport, operationLog) => {
        const reported = roundReport.confirmedOperations || [];
        const actual = operationLog.filter(op => op.from === roundReport.agentId);

        // 每个报告的操作必须有对应的实际操作记录
        for (let op of reported) {
          const found = actual.find(a =>
            a.operationId === op.operationId && a.success === op.success
          );
          if (!found) {
            return {
              valid: false,
              violation: "reported_operation_not_found",
              operationId: op.operationId
            };
          }
        }
        return { valid: true };
      }
    },

    // C2: 决策报告必须包含阈值计算
    decisionReportRequired: {
      validate: (roundReport, agentState) => {
        const decision = roundReport.decisionReport;
        if (!decision) {
          return { valid: false, violation: "decision_report_missing" };
        }

        // 检查必要字段
        const required = ["threshold", "candidates", "selectedDirection", "selectionReason"];
        for (let field of required) {
          if (!decision[field]) {
            return { valid: false, violation: `decision_report_missing_${field}` };
          }
        }

        // 验证计算正确性
        for (let candidate of decision.candidates) {
          const expectedProb = Math.pow(candidate.concentration, 2) /
            (Math.pow(candidate.concentration, 2) + Math.pow(decision.threshold, 2));

          if (Math.abs(candidate.responseProb - expectedProb) > 0.01) {
            return {
              valid: false,
              violation: "response_prob_calculation_error",
              expected: expectedProb,
              actual: candidate.responseProb
            };
          }
        }

        return { valid: true };
      }
    },

    // C3: 冲突审查报告必须存在
    conflictReviewRequired: {
      validate: (roundReport, blackboardSnapshot) => {
        const review = roundReport.conflictReview;
        if (!review) {
          return { valid: false, violation: "conflict_review_missing" };
        }

        // 检查是否真的审查了所有发现
        const findingIds = blackboardSnapshot.findings.map(f => f.id);
        const reviewedIds = review.reviewedFindings || [];

        const missing = findingIds.filter(id => !reviewedIds.includes(id));
        if (missing.length > 0) {
          return {
            valid: false,
            violation: "incomplete_conflict_review",
            missingFindings: missing
          };
        }

        return { valid: true };
      }
    },

    // C4: 随机探索必须执行
    randomExploreEnforced: {
      validate: (roundReport, instructions) => {
        if (instructions.forceRandomExplore) {
          if (!roundReport.randomExploreForced) {
            return { valid: false, violation: "random_explore_not_executed" };
          }
          // 检查是否真的选择了随机方向（不是最高浓度）
          if (roundReport.decisionReport?.selectedDirection ===
              roundReport.decisionReport?.candidates[0]?.direction) {
            return { valid: false, violation: "random_explore_fake" };
          }
        }
        return { valid: true };
      }
    }
  },

  // 执行检查
  runChecks: (roundReport, context) => {
    const results = [];

    for (let [checkName, check] of Object.entries(AGENT_COMPLIANCE_CHECKER.checks)) {
      const result = check.validate(roundReport, context);
      results.push({
        check: checkName,
        ...result
      });
    }

    return {
      agentId: roundReport.agentId,
      round: roundReport.round,
      compliant: results.every(r => r.valid),
      violations: results.filter(r => !r.valid),
      results
    };
  }
};
```

### 3.2 违规处理机制

```javascript
const VIOLATION_HANDLER = {
  // 违规等级
  levels: {
    WARNING: { score: 1, action: "record_only" },
    MINOR: { score: 3, action: "reduce_pheromone_weight" },
    MAJOR: { score: 5, action: "degrade_agent" },
    CRITICAL: { score: 10, action: "force_terminate" }
  },

  // 违规映射
  violationSeverity: {
    "reported_operation_not_found": "MAJOR",
    "decision_report_missing": "MAJOR",
    "decision_report_missing_threshold": "MINOR",
    "response_prob_calculation_error": "MINOR",
    "conflict_review_missing": "MINOR",
    "incomplete_conflict_review": "WARNING",
    "random_explore_not_executed": "MAJOR",
    "random_explore_fake": "MAJOR"
  },

  // 处理违规
  handle: async (blackboard, agentId, violations) => {
    const state = blackboard.metadata.agentStates[agentId];
    state.violations = state.violations || [];
    state.violationScore = state.violationScore || 0;

    for (let violation of violations) {
      const severity = VIOLATION_HANDLER.violationSeverity[violation.violation] || "WARNING";
      const level = VIOLATION_HANDLER.levels[severity];

      state.violations.push({
        ...violation,
        severity,
        timestamp: Date.now()
      });

      state.violationScore += level.score;

      console.log(`[COMPLIANCE] ${agentId} ${severity}: ${violation.violation}`);

      // 执行对应动作
      if (level.action === "degrade_agent" && state.status === "active") {
        state.status = "degraded";
        console.log(`[COMPLIANCE] ${agentId} 已降级`);
      }

      if (level.action === "force_terminate" || state.violationScore >= 15) {
        state.status = "terminated";
        state.terminationReason = "compliance_violation";
        console.log(`[COMPLIANCE] ${agentId} 已强制终止`);
      }
    }

    return state;
  }
};
```

---

## 四、运行目录与文档生成

### 4.1 目录创建脚本

```javascript
const RUN_DIRECTORY_MANAGER = {
  // 创建运行目录
  createRunDirectory: async (taskName) => {
    const date = new Date().toISOString().slice(0, 10);
    const taskSlug = taskName.toLowerCase()
      .replace(/[^a-z0-9\u4e00-\u9fff]+/g, '-')
      .slice(0, 30);
    const runId = `${date}-${taskSlug}`;
    const runPath = `K:/vscode/营销文档/swarm-runs/${runId}`;

    // 创建目录结构
    await Bash(`mkdir -p "${runPath}/agent-reports"`);

    return { runId, runPath };
  },

  // 保存运行配置
  saveRunConfig: async (runPath, config) => {
    const runConfig = {
      runId: config.runId,
      task: config.task,
      createdAt: Date.now(),
      swarmVersion: "v2.4",
      agents: config.agents.map(a => ({
        name: a.name,
        displayName: a.displayName,
        threshold: a.internalThreshold,
        randomExploreProb: a.randomExploreProb
      })),
      config: {
        maxRounds: 10,
        minRounds: 3,
        betaStability: 2,
        quorumThreshold: 0.67,
        minDiversity: 0.4,
        evaporationRate: 0.08
      }
    };

    await writeFile(`${runPath}/run-config.json`, JSON.stringify(runConfig, null, 2));
  },

  // 保存黑板状态
  saveBlackboard: async (runPath, blackboard) => {
    await writeFile(
      `${runPath}/blackboard.json`,
      JSON.stringify(blackboard, null, 2)
    );
  },

  // 保存操作日志
  saveOperationLog: async (runPath, operations) => {
    await writeFile(
      `${runPath}/operation-log.json`,
      JSON.stringify(operations, null, 2)
    );
  },

  // 保存Agent报告
  saveAgentReport: async (runPath, round, agentId, report) => {
    const roundDir = `${runPath}/agent-reports/round-${round}`;
    await Bash(`mkdir -p "${roundDir}"`);

    const reportContent = `# ${agentId} - Round ${round}

## 决策报告
\`\`\`json
${JSON.stringify(report.decisionReport, null, 2)}
\`\`\`

## 探索方向
${report.direction}

## 核心发现
${report.findings.map(f => `### ${f.coreIdea}
- **视角**: ${f.perspective}
- **详情**: ${f.details}
`).join('\n')}

## 已确认操作
| 操作 | 参数 | 结果 |
|------|------|------|
${report.confirmedOperations.map(op =>
  `| ${op.operation} | ${JSON.stringify(op.params)} | ${op.success ? '✅' : '❌'} |`
).join('\n')}

## 冲突审查
\`\`\`json
${JSON.stringify(report.conflictReview, null, 2)}
\`\`\`

---
*生成时间: ${new Date().toISOString()}*
`;

    await writeFile(`${roundDir}/${agentId}.md`, reportContent);
  },

  // 生成收敛报告
  generateConvergenceReport: async (runPath, blackboard, convergenceResult) => {
    const report = `# 收敛报告

## 收敛状态

| 指标 | 状态 | 数值 | 阈值 |
|------|------|------|------|
| β稳定性 | ${convergenceResult.betaStability.stable ? '✅' : '❌'} | ${convergenceResult.betaStability.rounds}轮 | 2轮 |
| 法定人数 | ${convergenceResult.quorum.quorum ? '✅' : '❌'} | ${(convergenceResult.quorum.quorumIdeas[0]?.supportRate * 100 || 0).toFixed(0)}% | 67% |
| 多样性 | ${convergenceResult.diversity.aboveThreshold ? '✅' : '❌'} | ${(convergenceResult.diversity.overall * 100).toFixed(0)}% | 40% |

## β稳定性详情
- **稳定轮数**: ${convergenceResult.betaStability.rounds}
- **共识观点**: ${convergenceResult.betaStability.consensus?.join(', ') || '无'}

## 法定人数详情
| 观点 | 支持者 | 支持率 |
|------|--------|--------|
${convergenceResult.quorum.quorumIdeas.map(idea =>
  `| ${idea.idea} | ${idea.supporters.join(', ')} | ${(idea.supportRate * 100).toFixed(0)}% |`
).join('\n')}

## 多样性详情
- **视角多样性**: ${(convergenceResult.diversity.perspectiveDiversity * 100).toFixed(0)}%
- **观点正交性**: ${(convergenceResult.diversity.orthogonality * 100).toFixed(0)}%
- **信息素熵**: ${(convergenceResult.diversity.entropy * 100).toFixed(0)}%

## Agent状态
| Agent | 角色 | 状态 | 违规分数 |
|-------|------|------|----------|
${Object.entries(blackboard.metadata.agentStates).map(([id, state]) =>
  `| ${id} | ${state.role} | ${state.status} | ${state.violationScore || 0} |`
).join('\n')}

---
*生成时间: ${new Date().toISOString()}*
`;

    await writeFile(`${runPath}/convergence-report.md`, report);
  },

  // 生成最终研究报告
  generateFinalReport: async (runPath, blackboard, convergenceResult, task) => {
    const findings = blackboard.metadata.findings;
    const pheromones = blackboard.metadata.pheromones;

    // 按信息素浓度排序方向
    const sortedDirections = Object.entries(pheromones)
      .sort((a, b) => b[1].concentration - a[1].concentration);

    // 提取共识观点
    const consensusIdeas = convergenceResult.quorum.quorumIdeas;

    // 提取创新观点（独特但高价值的）
    const innovativeFindings = findings.filter(f =>
      !consensusIdeas.some(ci => ci.idea === f.coreIdea)
    ).slice(0, 5);

    const report = `# ${task} 研究报告

> Swarm v2.4 协作研究成果

## 元信息

| 项目 | 值 |
|------|-----|
| **任务** | ${task} |
| **运行日期** | ${new Date().toISOString().slice(0, 10)} |
| **运行目录** | ${runPath} |
| **Swarm版本** | v2.4 |
| **Agent数量** | ${Object.keys(blackboard.metadata.agentStates).length} |
| **迭代轮数** | ${blackboard.metadata.currentRound || 3} |

## 执行摘要

${generateExecutiveSummary(consensusIdeas, sortedDirections)}

## 核心发现

${sortedDirections.slice(0, 5).map(([dir, ph], i) => `
### 发现${i + 1}: ${dir}

- **信息素浓度**: ${(ph.concentration * 100).toFixed(0)}%
- **沉积次数**: ${ph.depositedBy?.length || 0}
- **相关发现**:
${findings.filter(f => f.coreIdea?.includes(dir.slice(0, 10)) ||
                       dir.includes(f.coreIdea?.slice(0, 10) || ''))
  .slice(0, 3)
  .map(f => `  - ${f.coreIdea} (${f.agentId})`)
  .join('\n') || '  - 暂无详细发现'}
`).join('\n')}

## 共识观点（法定人数≥67%）

${consensusIdeas.map((idea, i) => `
### ${i + 1}. ${idea.idea}
- **支持率**: ${(idea.supportRate * 100).toFixed(0)}%
- **支持Agent**: ${idea.supporters.join(', ')}
`).join('\n') || '暂无达成共识的观点'}

## 创新观点

${innovativeFindings.map((f, i) => `
### ${i + 1}. ${f.coreIdea}
- **提出者**: ${f.agentId}
- **视角**: ${f.perspective}
- **详情**: ${f.details?.slice(0, 200)}${(f.details?.length || 0) > 200 ? '...' : ''}
`).join('\n') || '暂无创新观点'}

## 信息素分布

\`\`\`
${sortedDirections.map(([dir, ph]) =>
  `${dir.slice(0, 20).padEnd(22)} ${'█'.repeat(Math.round(ph.concentration * 20))} ${(ph.concentration * 100).toFixed(0)}%`
).join('\n')}
\`\`\`

## 研究方法说明

### 探索过程
- **探索轮次**: ${blackboard.metadata.currentRound || 3}
- **收敛条件**: β稳定性(2轮) + 法定人数(67%) + 多样性(40%)

### Agent角色演化
\`\`\`
${generateRoleEvolution(blackboard.metadata.agentStates)}
\`\`\`

### 合规性统计
- **总操作数**: ${blackboard.metadata.operationLog?.length || 0}
- **成功操作**: ${(blackboard.metadata.operationLog || []).filter(op => op.success).length}
- **违规Agent**: ${Object.values(blackboard.metadata.agentStates).filter(s => (s.violationScore || 0) > 0).length}

## 附录

- [Agent报告目录](./agent-reports/)
- [黑板完整状态](./blackboard.json)
- [操作日志](./operation-log.json)
- [收敛报告](./convergence-report.md)

---

*本报告由 Swarm v2.4 自动生成 | ${new Date().toISOString()}*
`;

    await writeFile(`${runPath}/final-research-report.md`, report);
    return `${runPath}/final-research-report.md`;
  }
};

// 辅助函数
function generateExecutiveSummary(consensusIdeas, directions) {
  if (consensusIdeas.length === 0) {
    return "本次协作未能达成共识，需要更多轮次的探索。";
  }

  const topDirection = directions[0];
  const consensusCount = consensusIdeas.length;

  return `经过多轮协作探索，Swarm系统识别出${directions.length}个潜在转型方向，
其中"${topDirection[0]}"方向获得最高信息素浓度(${(topDirection[1].concentration * 100).toFixed(0)}%)。
最终达成${consensusCount}项共识观点，覆盖了核心转型策略的主要维度。`;
}

function generateRoleEvolution(agentStates) {
  const lines = [];
  for (let [agentId, state] of Object.entries(agentStates)) {
    const history = state.roleHistory || [];
    if (history.length > 0) {
      const path = history.map(h => `${h.from}→${h.to}`).join(' ');
      lines.push(`${agentId}: EXPLORER ${path}`);
    } else {
      lines.push(`${agentId}: ${state.role} (无转换)`);
    }
  }
  return lines.join('\n');
}
```

---

## 五、完整执行流程（v2.4）

### 5.1 初始化阶段

```javascript
async function initializeSwarm(task, options = {}) {
  // 1. 创建运行目录
  const { runId, runPath } = await RUN_DIRECTORY_MANAGER.createRunDirectory(task);

  console.log(`[SWARM v2.4] 运行目录: ${runPath}`);

  // 2. 创建团队
  await TeamCreate({
    team_name: `swarm-${runId}`,
    description: `Swarm v2.4: ${task}`
  });

  // 3. 创建黑板
  const blackboardTask = await TaskCreate({
    subject: `Swarm黑板: ${task.slice(0, 50)}`,
    description: `## 共享黑板 v2.4\n\n### 操作方式\n通过blackboard_operation消息操作`,
    metadata: {
      runId,
      runPath,
      task,
      pheromones: {},
      findings: [],
      stopSignals: [],
      claims: {},
      opinionHistory: [],
      agentStates: {},
      operationLog: [],
      events: [],
      config: {
        version: "v2.4",
        evaporationRate: 0.08,
        depositAmount: 0.1,
        maxRounds: options.maxRounds || 10,
        minRounds: 3,
        betaStability: 2,
        quorumThreshold: 0.67,
        minDiversity: 0.4,
        shutdownTimeout: 30000
      }
    }
  });

  // 4. 初始化Agent状态
  const agentCount = options.agents || 4;
  const agents = [];
  const AGENT_NAMES = ["TanWei", "SuYuan", "DongCha", "QiuSuo"];

  for (let i = 0; i < agentCount; i++) {
    const name = AGENT_NAMES[i];
    const state = {
      role: "EXPLORER",
      displayName: AGENT_DISPLAY_NAMES[name],
      internalThreshold: 0.3 + Math.random() * 0.3,
      randomExploreProb: 0.1 + Math.random() * 0.1,
      status: "active",
      createdAt: Date.now(),
      lastActiveAt: Date.now(),
      stats: {
        pheromoneDeposits: 0,
        explorationRounds: 0,
        findingsCount: 0,
        signalsSent: 0
      },
      violations: [],
      violationScore: 0
    };

    blackboardTask.metadata.agentStates[name] = state;
    agents.push({ name, state });
  }

  // 5. 保存运行配置
  await RUN_DIRECTORY_MANAGER.saveRunConfig(runPath, {
    runId,
    task,
    agents
  });

  // 6. 启动Agent
  for (let agent of agents) {
    await Task({
      subagent_type: "Explore",
      name: agent.name,
      team_name: `swarm-${runId}`,
      prompt: generateAgentPrompt(agent.name, agent.state, task, blackboardTask.id)
    });
  }

  return {
    runId,
    runPath,
    blackboardId: blackboardTask.id,
    agents
  };
}
```

### 5.2 Agent Prompt生成器（v2.4强制版）

```javascript
function generateAgentPrompt(agentName, agentState, task, blackboardId) {
  return `你是Swarm v2.4的Explorer，名字是 **${agentName}**（${agentState.displayName}）。

## 任务
${task}

## 你的内在状态
\`\`\`json
{
  "role": "${agentState.role}",
  "displayName": "${agentState.displayName}",
  "internalThreshold": ${agentState.internalThreshold.toFixed(2)},
  "randomExploreProb": ${agentState.randomExploreProb.toFixed(2)}
}
\`\`\`

## 【v2.4 强制】黑板操作协议

### ⚠️ 核心规则
所有黑板操作**必须**通过SendMessage发送，**禁止**在报告中声明！

### 操作流程（严格执行）

**Step 1: 发送操作请求**
\`\`\`javascript
SendMessage({
  type: "blackboard_operation",
  operation: "deposit_pheromone",  // 或其他操作
  params: { direction: "方向描述", amount: 0.1 },
  summary: "沉积信息素: 方向描述"
})
\`\`\`

**Step 2: 等待确认**
发送后**必须等待** \`operation_result\` 返回才能继续。

**Step 3: 在round_complete中报告已确认的操作**
\`\`\`json
{
  "confirmedOperations": [
    { "operationId": "op-xxx", "operation": "deposit_pheromone", "success": true }
  ]
}
\`\`\`

### 可用操作

| 操作 | 参数 | 用途 |
|------|------|------|
| deposit_pheromone | direction, amount | 沉积信息素 |
| send_stop_signal | targetDirection, reason, evidence | 发送停止信号 |
| update_finding | finding | 更新发现 |
| claim_subtask | description | 声明子任务 |

## 【v2.4 强制】阈值响应决策协议

### 你的阈值
- **internalThreshold**: ${agentState.internalThreshold.toFixed(2)}
- **randomExploreProb**: ${agentState.randomExploreProb.toFixed(2)}

### 决策计算（每轮必须执行）

**公式**: P(S, θ) = S² / (S² + θ²)

**计算步骤**:
1. 从round_start获取各方向浓度S
2. 对每个方向计算响应概率P(S, ${agentState.internalThreshold.toFixed(2)})
3. 生成随机数r，如果r < P则选择该方向

**报告格式**（必须包含）:
\`\`\`json
{
  "decisionReport": {
    "threshold": ${agentState.internalThreshold.toFixed(2)},
    "candidates": [
      { "direction": "方向A", "concentration": 0.75, "responseProb": 0.80 }
    ],
    "selectedDirection": "方向A",
    "selectionReason": "最高响应概率0.80",
    "randomNumber": 0.42,
    "decision": "选择(0.42 < 0.80)"
  }
}
\`\`\`

## 【v2.4 强制】冲突审查协议

### 每轮必须执行
1. 审查blackboardSnapshot中的所有findings
2. 检测冲突、逻辑漏洞、证据不足
3. 发现问题时**必须**发送停止信号
4. 无问题也**必须**提交审查报告

**发送停止信号**:
\`\`\`javascript
SendMessage({
  type: "blackboard_operation",
  operation: "send_stop_signal",
  params: {
    targetDirection: "被质疑方向",
    reason: "contradictory_evidence",
    evidence: "具体证据"
  }
})
\`\`\`

**无冲突声明**:
\`\`\`json
{
  "conflictReview": {
    "reviewedFindings": ["finding-001", "finding-002"],
    "conflictsFound": 0,
    "stopSignalsSent": [],
    "declaration": "已审查所有发现，未发现可质疑的冲突"
  }
}
\`\`\`

## 【v2.4 强制】随机探索协议

如果round_start中 \`instructions.forceRandomExplore: true\`:
- **必须**忽略所有响应概率
- **必须**随机选择方向
- **必须**在报告中标记 \`"randomExploreForced": true\`

## round_complete报告格式

\`\`\`json
{
  "type": "round_complete",
  "round": N,
  "direction": "探索方向",
  "decisionReport": { /* 决策报告 */ },
  "findings": [{ "coreIdea": "观点", "perspective": "视角", "details": "详情" }],
  "confirmedOperations": [ /* 已确认的操作 */ ],
  "conflictReview": { /* 冲突审查 */ },
  "randomExploreForced": false
}
\`\`\`

## 合规性警告
- 违规操作将导致降级或终止
- 连续2轮无冲突审查 = 违规
- 虚假报告 = 违规
- 违规分数≥15 = 强制终止

---
立即开始探索任务。等待round_start消息。
`;
}
```

---

## 六、验证清单

### 6.1 协议验证

| # | 检查项 | v2.3 | v2.4目标 |
|---|--------|------|----------|
| 1 | 黑板操作通过SendMessage | ❌ | ✅ |
| 2 | 操作返回operation_result | ❌ | ✅ |
| 3 | 决策报告包含阈值计算 | ❌ | ✅ |
| 4 | 冲突审查报告存在 | ❌ | ✅ |
| 5 | 随机探索实际执行 | ❌ | ✅ |
| 6 | 自动强制终止 | ❌ | ✅ |
| 7 | 运行目录独立 | ❌ | ✅ |
| 8 | 研究文档生成 | ❌ | ✅ |

### 6.2 输出验证

每次运行必须生成：
- [ ] `run-config.json` - 运行配置
- [ ] `blackboard.json` - 黑板最终状态
- [ ] `operation-log.json` - 操作日志
- [ ] `agent-reports/round-N/*.md` - Agent报告
- [ ] `convergence-report.md` - 收敛报告
- [ ] `final-research-report.md` - 最终研究成果

---

## 七、版本历史

| 版本 | 日期 | 主要改进 |
|------|------|----------|
| v2.3 | 2026-02-18 | 执行层修复（未完全生效） |
| **v2.4** | 2026-02-18 | **强制协议执行** + **独立运行目录** + **研究文档生成** |

---

*Swarm v2.4 升级方案 | 2026-02-18*
