# Swarm v2.3 升级方案

> 基于v2.2实际运行复盘，修复"协议存在但未执行"的根本问题

**版本**: v2.3.0
**日期**: 2026-02-18
**基于**: Swarm v2.2 卷烟零售价值研究 实际运行复盘
**状态**: ✅ 已应用到 skill.md

---

## 一、核心发现：协议存在但未执行

### 1.1 问题本质

v2.2的skill.md已经包含了完整的协议定义，但**Orchestrator在执行时没有遵循**。

**表现**:
- 协议文档写得很完整 ✓
- Orchestrator理解协议 ✓
- 但执行时跳过了关键步骤 ✗

### 1.2 实际运行中发现的问题

| # | 问题 | 优先级 | 符合度 | 根本原因 |
|---|------|--------|--------|----------|
| **P1** | 黑板代理机制未执行 | P0 | 0% | 收到blackboard_operation但不处理 |
| **P2** | 信息素系统是假的 | P0 | 30% | 浓度手动设置，非Agent驱动 |
| **P3** | Shutdown流程失败 | P0 | 失败 | 三阶段协议未真正执行 |
| **P4** | 阈值响应模型未应用 | P1 | 10% | 公式存在但决策时不用 |
| **P5** | 角色转换通知不完整 | P1 | 50% | 转换了但没发消息 |
| **P6** | 收敛判断不准确 | P1 | 30% | 声称收敛但未真正计算 |
| **P7** | Agent沉默（QiuSuo） | P1 | N/A | 某Agent几乎无输出 |
| **P8** | 多样性保护未验证 | P2 | 30% | 没有实际计算多样性 |

---

## 二、根本原因分析

### 2.1 Orchestrator执行缺失

```
理论流程:
Agent → blackboard_operation → Orchestrator处理 → 更新黑板 → 返回operation_result

实际流程:
Agent → blackboard_operation → Orchestrator(收到但不处理) → 无后续

问题: Orchestrator"知道"要做什么，但"没有做"
```

### 2.2 信息素系统失真

```
理论流程:
Agent发现价值 → deposit_pheromone → 浓度增加 → 引导其他Agent

实际流程:
Orchestrator手动设置 → 浓度值变成"装饰" → Agent决策不受影响

问题: 信息素变成了"记录"，而非"驱动"
```

### 2.3 Shutdown协议形式化

```
理论流程:
预通知(5s) → 优雅关闭(15s) → 强制终止(10s) → 标记terminated

实际流程:
发送shutdown_request → 等待响应 → 无响应 → 放弃

问题: 第三阶段"强制终止"未执行
```

---

## 三、v2.2.1 升级方案

### 3.1 P0: Orchestrator执行检查清单（核心修复）

**问题**: Orchestrator理解协议但不执行

**解决方案**: 引入**强制执行检查清单**，每一步必须显式确认

```javascript
// 新增: Orchestrator执行状态机
const ORCHESTRATOR_EXECUTION_STATE = {
  currentPhase: "INIT",
  completedSteps: new Set(),
  requiredSteps: {
    "INIT": ["create_team", "create_blackboard", "init_agent_states", "spawn_agents"],
    "ROUND": ["broadcast_round_start", "wait_responses", "process_operations", "update_blackboard", "return_results", "settle_round"],
    "CONVERGENCE": ["check_beta_stability", "check_quorum", "check_diversity", "generate_report"],
    "SHUTDOWN": ["pre_notify", "graceful_request", "force_terminate", "cleanup"]
  }
};

// 强制检查函数
function assertStepCompleted(phase, step) {
  if (!ORCHESTRATOR_EXECUTION_STATE.completedSteps.has(`${phase}:${step}`)) {
    throw new Error(`CRITICAL: Step ${phase}:${step} not completed!`);
  }
}

// 标记完成
function markStepCompleted(phase, step) {
  ORCHESTRATOR_EXECUTION_STATE.completedSteps.add(`${phase}:${step}`);
  console.log(`[CHECKLIST] ✓ ${phase}:${step}`);
}
```

**实施**: 在Orchestrator每一步操作后必须调用`markStepCompleted`，在进入下一阶段前必须调用`assertStepCompleted`

---

### 3.2 P0: 黑板代理强制执行

**问题**: Agent发送`blackboard_operation`但Orchestrator不处理

**解决方案**: 引入**操作队列**和**确认机制**

```javascript
// 新增: 黑板操作队列
const BLACKBOARD_OPERATION_QUEUE = {
  pending: [],
  processed: [],
  failed: []
};

// 新增: 操作处理函数（必须执行）
async function processBlackboardOperation(message) {
  const { operation, params } = message;
  const agentId = message.from;

  // 1. 记录操作请求
  const opRecord = {
    id: `op-${Date.now()}`,
    operation,
    params,
    from: agentId,
    status: "pending",
    timestamp: Date.now()
  };
  BLACKBOARD_OPERATION_QUEUE.pending.push(opRecord);

  // 2. 执行操作
  const opDef = BLACKBOARD_OPERATIONS[operation];
  if (!opDef) {
    opRecord.status = "failed";
    opRecord.error = "unknown_operation";
    BLACKBOARD_OPERATION_QUEUE.failed.push(opRecord);
    return { success: false, error: "unknown_operation" };
  }

  const result = opDef.handler(blackboard, params, agentId);
  opRecord.status = "completed";
  opRecord.result = result;
  BLACKBOARD_OPERATION_QUEUE.processed.push(opRecord);

  // 3. 【关键】必须返回operation_result给Agent
  await SendMessage({
    type: "message",
    recipient: agentId,
    content: JSON.stringify({
      type: "operation_result",
      operationId: opRecord.id,
      ...result
    }),
    summary: `黑板操作结果: ${operation} - ${result.success ? '成功' : '失败'}`
  });

  // 4. 更新黑板
  await TaskUpdate({
    taskId: blackboardTaskId,
    metadata: {
      pheromones: blackboard.metadata.pheromones,
      findings: blackboard.metadata.findings,
      stopSignals: blackboard.metadata.stopSignals,
      agentStates: blackboard.metadata.agentStates,
      operationLog: BLACKBOARD_OPERATION_QUEUE.processed
    }
  });

  return result;
}

// 检查清单验证
function validateBlackboardOperations() {
  const pending = BLACKBOARD_OPERATION_QUEUE.pending.length;
  const processed = BLACKBOARD_OPERATION_QUEUE.processed.length;

  if (pending > processed) {
    console.error(`CHECKLIST FAILED: ${pending - processed} operations not processed`);
    return false;
  }
  return true;
}
```

**Orchestrator Prompt 更新（强制提醒）**:

```markdown
## 【v2.2.1 强制执行】黑板代理机制

### 收到Agent消息时的处理流程（必须执行）

1. **解析消息**: 检查消息类型
2. **如果是blackboard_operation**:
   - 立即调用 `processBlackboardOperation()`
   - **必须**返回 `operation_result` 给Agent
   - **必须**使用 TaskUpdate 更新黑板 metadata
3. **记录到执行日志**: 标记该操作已完成

### ⚠️ 常见错误（必须避免）

- ❌ 收到blackboard_operation但不处理
- ❌ 处理了但没返回operation_result
- ❌ 返回了但没更新黑板Task的metadata
- ❌ 更新了黑板但Agent状态没同步

### 验证方法

每轮结束时检查:
```
pending_operations = BLACKBOARD_OPERATION_QUEUE.pending.length
processed_operations = BLACKBOARD_OPERATION_QUEUE.processed.length
assert(pending_operations === processed_operations, "有操作未处理!")
```
```

---

### 3.3 P0: 信息素系统真实化

**问题**: 信息素浓度手动设置，非Agent驱动

**解决方案**: 信息素**只能**通过Agent的`deposit_pheromone`操作修改

```javascript
// 修改: 信息素操作（唯一入口）
const BLACKBOARD_OPERATIONS = {
  DEPOSIT_PHEROMONE: {
    name: "deposit_pheromone",
    params: {
      direction: "string",
      amount: "number"  // 可选，默认0.1
    },
    handler: (blackboard, params, agentId) => {
      const ph = blackboard.metadata.pheromones;
      const direction = params.direction;
      const amount = params.amount || 0.1;

      // 【关键】这是唯一修改信息素浓度的方式
      if (!ph[direction]) {
        ph[direction] = {
          concentration: 0,
          depositedBy: [],
          createdAt: Date.now()
        };
      }

      // 沉积（累加）
      ph[direction].concentration = Math.min(
        ph[direction].concentration + amount,
        1.0
      );
      ph[direction].lastUpdate = Date.now();
      ph[direction].lastDepositor = agentId;

      if (!ph[direction].depositedBy.includes(agentId)) {
        ph[direction].depositedBy.push(agentId);
      }

      // 更新Agent统计
      blackboard.metadata.agentStates[agentId].stats.pheromoneDeposits++;

      return {
        success: true,
        direction: direction,
        newConcentration: ph[direction].concentration,
        depositedBy: agentId
      };
    }
  }
};

// 新增: 信息素蒸发（轮次结算时自动执行）
function evaporatePheromones(blackboard, rate = 0.08) {
  const ph = blackboard.metadata.pheromones;

  for (let direction in ph) {
    const oldConc = ph[direction].concentration;
    ph[direction].concentration = Math.max(
      oldConc * (1 - rate),
      0.1  // 最低阈值
    );

    // 记录蒸发事件
    ph[direction].evaporationHistory = ph[direction].evaporationHistory || [];
    ph[direction].evaporationHistory.push({
      round: blackboard.metadata.currentRound,
      before: oldConc,
      after: ph[direction].concentration,
      timestamp: Date.now()
    });
  }
}

// 禁止: 直接设置信息素浓度
// ❌ blackboard.metadata.pheromones["方向A"].concentration = 0.8;
// ✅ 必须通过Agent的deposit_pheromone操作
```

**Orchestrator约束**:

```markdown
## 【v2.2.1 信息素规则】

### 禁止行为
- ❌ 直接设置信息素浓度
- ❌ 在轮次开始时预设浓度
- ❌ 根据报告内容"推断"浓度

### 允许行为
- ✅ 在settleRound时执行蒸发
- ✅ 响应Agent的deposit_pheromone操作
- ✅ 记录信息素变化历史

### 初始状态
所有方向信息素浓度为0，等待Agent探索后沉积。
```

---

### 3.4 P0: Shutdown强制终止机制

**问题**: Agent不响应shutdown_request时没有强制终止

**解决方案**: 实现**真正的三阶段关闭**，第三阶段必须执行

```javascript
// 新增: 强制终止能力
const FORCE_TERMINATE_PROTOCOL = {
  // 阶段3: 强制终止
  execute: async (agentId, blackboard) => {
    console.warn(`[FORCE TERMINATE] Agent ${agentId} 强制终止`);

    // 1. 标记Agent状态
    if (blackboard.metadata.agentStates[agentId]) {
      blackboard.metadata.agentStates[agentId].status = "terminated";
      blackboard.metadata.agentStates[agentId].terminatedAt = Date.now();
      blackboard.metadata.agentStates[agentId].terminationReason = "forced";
      blackboard.metadata.agentStates[agentId].terminationRound = blackboard.metadata.currentRound;
    }

    // 2. 从活跃列表移除
    const teamConfig = await readTeamConfig();
    teamConfig.members = teamConfig.members.filter(m => m.name !== agentId);
    await writeTeamConfig(teamConfig);

    // 3. 记录终止事件
    blackboard.metadata.events = blackboard.metadata.events || [];
    blackboard.metadata.events.push({
      type: "agent_force_terminated",
      agentId: agentId,
      round: blackboard.metadata.currentRound,
      timestamp: Date.now()
    });

    return { terminated: true, agentId };
  }
};

// 完整三阶段关闭
async function executeShutdown(agents, blackboard) {
  const results = { graceful: [], forced: [], failed: [] };

  // 阶段1: 预通知（5秒）
  console.log("[SHUTDOWN] Phase 1: Pre-notify");
  for (let agent of agents) {
    await SendMessage({
      type: "message",
      recipient: agent.id,
      content: JSON.stringify({
        type: "shutdown_imminent",
        reason: "协作完成",
        prepareTime: 5000
      }),
      summary: "预通知: 协作即将结束"
    });
  }
  await sleep(5000);
  markStepCompleted("SHUTDOWN", "pre_notify");

  // 阶段2: 优雅关闭请求（15秒）
  console.log("[SHUTDOWN] Phase 2: Graceful request");
  for (let agent of agents) {
    const startTime = Date.now();
    try {
      await SendMessage({
        type: "shutdown_request",
        recipient: agent.id,
        content: "请关闭"
      });

      // 等待响应（最多15秒）
      const response = await waitForShutdownResponse(agent.id, 15000);
      if (response && response.acknowledged) {
        results.graceful.push(agent.id);
        console.log(`[SHUTDOWN] ✓ ${agent.id} 优雅关闭`);
      }
    } catch (e) {
      console.log(`[SHUTDOWN] ✗ ${agent.id} 无响应`);
    }
  }
  markStepCompleted("SHUTDOWN", "graceful_request");

  // 阶段3: 强制终止（必须执行）
  console.log("[SHUTDOWN] Phase 3: Force terminate");
  const remainingAgents = agents.filter(a =>
    !results.graceful.includes(a.id)
  );

  if (remainingAgents.length > 0) {
    console.warn(`[SHUTDOWN] ⚠️ ${remainingAgents.length} 个Agent需要强制终止`);

    for (let agent of remainingAgents) {
      await FORCE_TERMINATE_PROTOCOL.execute(agent.id, blackboard);
      results.forced.push(agent.id);
      console.log(`[SHUTDOWN] ⚡ ${agent.id} 强制终止`);
    }
  }
  markStepCompleted("SHUTDOWN", "force_terminate");

  // 更新黑板
  await TaskUpdate({
    taskId: blackboardTaskId,
    metadata: {
      ...blackboard.metadata,
      shutdownResult: results
    }
  });

  return results;
}
```

**Orchestrator Prompt 更新**:

```markdown
## 【v2.2.1 Shutdown协议】

### 三阶段必须全部执行

| 阶段 | 超时 | 动作 | 必须执行 |
|------|------|------|----------|
| 1. 预通知 | 5秒 | broadcast shutdown_imminent | ✓ |
| 2. 优雅关闭 | 15秒 | send shutdown_request | ✓ |
| 3. 强制终止 | 立即 | 标记terminated，更新黑板 | ✓✓✓ |

### ⚠️ 关键：第三阶段不可跳过

即使所有Agent都响应了优雅关闭，也要执行第三阶段：
1. 更新所有Agent状态为terminated
2. 更新黑板shutdownResult
3. 确认TeamConfig.members已清空

### 验证

```
assert(
  Object.values(agentStates).every(s => s.status === "terminated"),
  "所有Agent必须终止"
);
```
```

---

### 3.5 P1: 阈值响应模型强制应用

**问题**: 阈值响应公式存在但决策时不使用

**解决方案**: 在Agent决策点**强制计算**响应概率

```javascript
// 阈值响应函数（理论核心）
function responseProbability(stimulus, threshold) {
  if (stimulus === 0) return 0;
  // P(S,θ) = S² / (S² + θ²)
  return (stimulus ** 2) / (stimulus ** 2 + threshold ** 2);
}

// Agent决策辅助函数
function calculateAgentDecision(blackboard, agentId, candidateDirections) {
  const state = blackboard.metadata.agentStates[agentId];
  const threshold = state.internalThreshold;

  const decisions = candidateDirections.map(dir => {
    const rawConc = blackboard.metadata.pheromones[dir]?.concentration || 0;

    // 检查停止信号抑制
    const inhibition = checkInhibition(blackboard, agentId, dir);
    const effectiveConc = inhibition.effectiveConcentration || rawConc;

    // 计算响应概率
    const responseProb = responseProbability(effectiveConc, threshold);

    return {
      direction: dir,
      rawConcentration: rawConc,
      effectiveConcentration: effectiveConc,
      responseProbability: responseProb,
      shouldExplore: Math.random() < responseProb,
      inhibition: inhibition
    };
  });

  // 返回决策结果（用于round_start消息）
  return {
    agentId,
    threshold,
    decisions,
    recommendation: decisions.sort((a, b) => b.responseProbability - a.responseProbability)[0]
  };
}

// 在broadcastRoundStart中使用
async function broadcastRoundStart(round, blackboard) {
  for (let agentId in blackboard.metadata.agentStates) {
    const state = blackboard.metadata.agentStates[agentId];
    if (state.status !== "active") continue;

    // 获取所有方向
    const directions = Object.keys(blackboard.metadata.pheromones);

    // 【关键】计算阈值响应决策
    const decision = calculateAgentDecision(blackboard, agentId, directions);

    // 决定是否强制随机探索
    const forceRandomExplore = Math.random() < state.randomExploreProb;

    await SendMessage({
      type: "message",
      recipient: agentId,
      content: JSON.stringify({
        type: "round_start",
        round: round,
        agentState: state,
        blackboardSnapshot: {
          pheromones: blackboard.metadata.pheromones,
          stopSignals: blackboard.metadata.stopSignals,
          findings: blackboard.metadata.findings
        },
        // 【v2.2.1 新增】决策辅助信息
        decisionSupport: {
          threshold: state.internalThreshold,
          topDirections: decision.decisions.slice(0, 3),
          forceRandomExplore: forceRandomExplore
        },
        instructions: {
          forceRandomExplore: forceRandomExplore,
          recommendedDirection: forceRandomExplore ? null : decision.recommendation.direction
        }
      }),
      summary: `Round ${round} 开始`
    });
  }
}
```

---

### 3.6 P1: 角色转换消息强制发送

**问题**: 角色转换执行了但没通知Agent

**解决方案**: 在`settleRound`中**必须**发送`role_transition_executed`消息

```javascript
async function settleRound(blackboard, currentRound) {
  // ... 其他结算逻辑 ...

  // 检查角色转换
  const transitions = checkRoleTransitions(blackboard);

  for (let transition of transitions) {
    const state = blackboard.metadata.agentStates[transition.agentId];
    const oldRole = state.role;

    // 执行转换
    state.role = transition.toRole;
    state.roleHistory = state.roleHistory || [];
    state.roleHistory.push({
      from: oldRole,
      to: transition.toRole,
      reason: transition.reason,
      round: currentRound,
      timestamp: Date.now()
    });

    // 【关键】立即发送通知（不可跳过）
    await SendMessage({
      type: "message",
      recipient: transition.agentId,
      content: JSON.stringify({
        type: "role_transition_executed",
        fromRole: oldRole,
        toRole: transition.toRole,
        reason: transition.reason,
        round: currentRound,
        capabilities: ROLE_CAPABILITIES[transition.toRole],
        instructions: getRoleInstructions(transition.toRole)
      }),
      summary: `角色转换: ${oldRole} → ${transition.toRole}`
    });

    console.log(`[ROLE TRANSITION] ${transition.agentId}: ${oldRole} → ${transition.toRole}`);
    markStepCompleted("ROUND", `role_transition_${transition.agentId}`);
  }

  // 记录转换日志
  if (transitions.length > 0) {
    blackboard.metadata.roleTransitionLog = blackboard.metadata.roleTransitionLog || [];
    blackboard.metadata.roleTransitionLog.push({
      round: currentRound,
      transitions: transitions,
      timestamp: Date.now()
    });
  }
}
```

---

### 3.7 P1: Agent沉默检测与恢复

**问题**: QiuSuo几乎无输出

**解决方案**: 引入**Agent沉默检测**和**强制唤醒**机制

```javascript
const AGENT_SILENCE_DETECTION = {
  maxSilenceTime: 60000,     // 60秒无响应视为沉默
  maxSilenceRounds: 2,       // 连续2轮沉默触发恢复

  // 检测沉默
  detectSilence: (blackboard, agentId) => {
    const state = blackboard.metadata.agentStates[agentId];
    if (!state) return { silent: false };

    const lastActive = state.lastActiveAt || 0;
    const silenceTime = Date.now() - lastActive;
    const silenceRounds = state.stats.silenceRounds || 0;

    return {
      silent: silenceTime > AGENT_SILENCE_DETECTION.maxSilenceTime,
      silenceTime,
      silenceRounds,
      needsRecovery: silenceRounds >= AGENT_SILENCE_DETECTION.maxSilenceRounds
    };
  },

  // 恢复沉默Agent
  recover: async (blackboard, agentId, currentRound) => {
    const state = blackboard.metadata.agentStates[agentId];

    console.log(`[AGENT RECOVERY] 尝试恢复 ${agentId}`);

    // 发送强制唤醒消息
    await SendMessage({
      type: "message",
      recipient: agentId,
      content: JSON.stringify({
        type: "force_wake",
        round: currentRound,
        message: "你已沉默多轮，请立即响应。选择任意方向开始探索。",
        blackboardSnapshot: {
          pheromones: blackboard.metadata.pheromones,
          findings: blackboard.metadata.findings
        },
        instructions: {
          forceRandomExplore: true,
          mustRespond: true
        }
      }),
      summary: `强制唤醒: ${agentId}`
    });

    // 更新状态
    state.lastActiveAt = Date.now();
    state.stats.recoveryAttempts = (state.stats.recoveryAttempts || 0) + 1;

    return { recovered: true, agentId };
  },

  // 降级无法恢复的Agent
  degrade: async (blackboard, agentId) => {
    const state = blackboard.metadata.agentStates[agentId];
    state.status = "degraded";
    state.degradedAt = Date.now();
    state.degradedReason = "prolonged_silence";

    console.warn(`[AGENT DEGRADE] ${agentId} 因长期沉默被降级`);

    return { degraded: true, agentId };
  }
};

// 在每轮检查
async function checkAndRecoverSilentAgents(blackboard, currentRound) {
  for (let agentId in blackboard.metadata.agentStates) {
    const state = blackboard.metadata.agentStates[agentId];
    if (state.status !== "active") continue;

    const silence = AGENT_SILENCE_DETECTION.detectSilence(blackboard, agentId);

    if (silence.needsRecovery) {
      if (state.stats.recoveryAttempts >= 3) {
        await AGENT_SILENCE_DETECTION.degrade(blackboard, agentId);
      } else {
        await AGENT_SILENCE_DETECTION.recover(blackboard, agentId, currentRound);
      }
    }
  }
}
```

---

### 3.8 P2: 收敛判断真实计算

**问题**: 声称收敛但未真正计算β稳定性和多样性

**解决方案**: 引入**收敛计算器**，必须输出计算过程

```javascript
const CONVERGENCE_CALCULATOR = {
  // β稳定性检查
  checkBetaStability: (blackboard, beta = 2) => {
    const history = blackboard.metadata.opinionHistory || [];

    if (history.length < beta) {
      return {
        stable: false,
        reason: `历史不足: ${history.length}/${beta}`,
        rounds: history.length
      };
    }

    // 提取最近β轮的核心观点集合
    const recentHistory = history.slice(-beta);
    const opinionSets = recentHistory.map(h =>
      new Set(h.findings.map(f => f.coreIdea).filter(Boolean))
    );

    // 检查集合是否相同
    const firstSet = opinionSets[0];
    const isStable = opinionSets.every(set =>
      setsEqual(set, firstSet)
    );

    return {
      stable: isStable,
      rounds: beta,
      opinionSets: opinionSets.map(s => Array.from(s)),
      consensus: Array.from(firstSet)
    };
  },

  // 法定人数检查
  checkQuorum: (blackboard, threshold = 0.67) => {
    const findings = blackboard.metadata.findings || [];
    const activeAgents = Object.values(blackboard.metadata.agentStates)
      .filter(s => s.status === "active").length;

    if (activeAgents === 0) {
      return { quorum: false, reason: "无活跃Agent" };
    }

    // 统计每个观点的支持人数
    const ideaSupport = {};
    findings.forEach(f => {
      if (f.coreIdea) {
        ideaSupport[f.coreIdea] = ideaSupport[f.coreIdea] || new Set();
        ideaSupport[f.coreIdea].add(f.agentId);
      }
    });

    // 找出达到法定人数的观点
    const quorumIdeas = [];
    for (let [idea, supporters] of Object.entries(ideaSupport)) {
      const rate = supporters.size / activeAgents;
      if (rate >= threshold) {
        quorumIdeas.push({
          idea,
          supporters: Array.from(supporters),
          supportRate: rate
        });
      }
    }

    return {
      quorum: quorumIdeas.length > 0,
      threshold,
      activeAgents,
      quorumIdeas,
      allIdeas: Object.keys(ideaSupport)
    };
  },

  // 多样性检查
  checkDiversity: (blackboard, minThreshold = 0.4) => {
    const findings = blackboard.metadata.findings || [];
    const pheromones = blackboard.metadata.pheromones || {};

    // 1. 视角多样性
    const perspectives = new Set(findings.map(f => f.perspective).filter(Boolean));
    const perspectiveDiversity = Math.min(perspectives.size / 6, 1);

    // 2. 观点正交性
    const ideas = findings.map(f => f.coreIdea).filter(Boolean);
    const uniqueIdeas = new Set(ideas);
    const orthogonality = uniqueIdeas.size / Math.max(ideas.length, 1);

    // 3. 信息素分布熵
    const concentrations = Object.values(pheromones).map(p => p.concentration || 0);
    const totalConc = concentrations.reduce((a, b) => a + b, 0);
    const entropy = totalConc > 0
      ? -concentrations.reduce((sum, c) => {
          const p = c / totalConc;
          return sum + (p > 0 ? p * Math.log2(p) : 0);
        }, 0) / Math.log2(Math.max(concentrations.length, 2))
      : 0;

    const overall = (perspectiveDiversity + orthogonality + entropy) / 3;

    return {
      perspectiveDiversity,
      orthogonality,
      entropy,
      overall,
      aboveThreshold: overall >= minThreshold,
      details: {
        perspectiveCount: perspectives.size,
        uniqueIdeaCount: uniqueIdeas.size,
        totalIdeaCount: ideas.length,
        directionCount: concentrations.length
      }
    };
  },

  // 综合收敛判断
  evaluate: (blackboard, currentRound, config = {}) => {
    const {
      minRounds = 3,
      betaStability = 2,
      quorumThreshold = 0.67,
      minDiversity = 0.4
    } = config;

    const result = {
      round: currentRound,
      minRoundsMet: currentRound >= minRounds,
      betaStability: null,
      quorum: null,
      diversity: null,
      converged: false,
      reason: ""
    };

    // 1. 最少轮数
    if (!result.minRoundsMet) {
      result.reason = `轮数不足: ${currentRound}/${minRounds}`;
      return result;
    }

    // 2. β稳定性
    result.betaStability = CONVERGENCE_CALCULATOR.checkBetaStability(blackboard, betaStability);
    if (!result.betaStability.stable) {
      result.reason = `观点不稳定: ${result.betaStability.reason}`;
      return result;
    }

    // 3. 法定人数
    result.quorum = CONVERGENCE_CALCULATOR.checkQuorum(blackboard, quorumThreshold);
    if (!result.quorum.quorum) {
      result.reason = `无法定人数共识`;
      return result;
    }

    // 4. 多样性
    result.diversity = CONVERGENCE_CALCULATOR.checkDiversity(blackboard, minDiversity);
    if (!result.diversity.aboveThreshold) {
      result.reason = `多样性不足: ${result.diversity.overall.toFixed(2)}/${minDiversity}`;
      return result;
    }

    // 全部通过
    result.converged = true;
    result.reason = "所有收敛条件满足";

    return result;
  }
};
```

---

## 四、Orchestrator执行检查清单（v2.2.1）

### 4.1 初始化阶段
- [ ] `create_team` - TeamCreate
- [ ] `create_blackboard` - TaskCreate（不设置owner）
- [ ] `init_agent_states` - 初始化所有Agent状态
- [ ] `spawn_agents` - 启动所有Explorer

### 4.2 每轮执行
- [ ] `broadcast_round_start` - 发送轮次开始（含决策辅助）
- [ ] `wait_responses` - 等待Agent响应
- [ ] `process_operations` - **处理所有blackboard_operation**
- [ ] `return_results` - **返回operation_result给每个Agent**
- [ ] `update_blackboard` - 使用TaskUpdate更新黑板
- [ ] `settle_round` - 信息素蒸发、清理信号
- [ ] `check_role_transitions` - 检查并**发送**角色转换消息
- [ ] `check_silent_agents` - 检查沉默Agent并恢复

### 4.3 收敛阶段
- [ ] `calculate_beta_stability` - 计算β稳定性
- [ ] `calculate_quorum` - 计算法定人数
- [ ] `calculate_diversity` - 计算多样性
- [ ] `generate_report` - 生成报告

### 4.4 关闭阶段
- [ ] `pre_notify` - 预通知（5秒）
- [ ] `graceful_request` - 优雅关闭请求（15秒）
- [ ] `force_terminate` - **强制终止未响应Agent**
- [ ] `update_blackboard_final` - 更新最终黑板状态
- [ ] `cleanup` - 清理资源

---

## 五、验证清单

升级后必须验证：

### P0 验证
1. **黑板代理**: Agent发送blackboard_operation后，收到operation_result
2. **信息素真实**: 信息素浓度变化历史可追溯，全部由Agent沉积
3. **Shutdown成功**: 所有Agent最终状态为terminated，TeamDelete成功

### P1 验证
4. **阈值响应**: Agent决策参考responseProbability计算结果
5. **角色转换消息**: 转换后Agent收到role_transition_executed消息
6. **收敛计算**: 收敛报告包含β稳定性、法定人数、多样性的具体数值
7. **沉默恢复**: 沉默Agent被检测并尝试恢复

### P2 验证
8. **多样性达标**: 收敛时diversity.overall ≥ 0.4

---

## 六、实施优先级

| 优先级 | 问题 | 实施复杂度 | 预期效果 |
|--------|------|-----------|----------|
| P0 | 黑板代理执行 | 中 | 核心机制恢复 |
| P0 | 信息素真实化 | 中 | 自组织基础 |
| P0 | Shutdown强制终止 | 低 | 资源清理 |
| P1 | 阈值响应应用 | 中 | 决策符合理论 |
| P1 | 角色转换消息 | 低 | Agent感知角色 |
| P1 | Agent沉默检测 | 低 | 提升可靠性 |
| P1 | 收敛真实计算 | 中 | 判断可信 |
| P2 | 多样性保护 | 低 | 探索充分 |

---

## 七、Skill.md 更新计划

### 需要更新的章节

1. **Orchestrator执行检查清单** - 新增强制性要求
2. **黑板代理机制** - 强调必须返回operation_result
3. **信息素系统** - 强调只能通过deposit_pheromone修改
4. **Shutdown协议** - 强调第三阶段必须执行
5. **阈值响应模型** - 新增decisionSupport字段
6. **收敛判断** - 新增CONVERGENCE_CALCULATOR

---

*Swarm v2.2.1 升级方案 | 2026-02-18*
*核心修复: 协议存在但未执行 → 强制执行检查清单*
