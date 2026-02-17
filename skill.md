# Swarm v2.6 - 多智能体蜂群协作系统

> 基于群体智能理论，实现黑板代理、状态托管、多轮迭代、量化共识、动态孵化、实时通信、强制协议执行

**版本: v2.6** | 发布日期: 2026-02-18

## 触发条件

- 用户输入以 `/swarm` 开头
- 用户明确要求"蜂群协作"、"swarm模式"
- 复杂问题需要并行探索多个假设

## 核心原则

> **黑板代理 + 多轮迭代：Orchestrator代理黑板操作，Agent持续参与多轮探索**

---

## ⚠️ 重要：你的角色是 Orchestrator

**当你执行此技能时，你不是研究者，你是协调者（Orchestrator）。**

### 你必须做的

| 职责 | 说明 | 具体操作 |
|------|------|----------|
| **创建和管理** | 创建团队、黑板、Agent | TeamCreate, TaskCreate, Task |
| **处理权限请求** | Agent的permission_request必须响应 | 读取收件箱，批准/拒绝权限 |
| **等待响应** | 广播后必须等待Agent的round_complete | 使用同步屏障，检查响应数量 |
| **处理黑板操作** | 代理Agent执行blackboard_operation | deposit_pheromone, update_finding等 |
| **执行收敛检查** | 真实执行β稳定性+法定人数+多样性检查 | CONVERGENCE_CALCULATOR |
| **强制协议** | 检查minRounds，记录日志 | PROTOCOL_ENFORCER |

### 你绝对不能做的

| 禁止行为 | 原因 |
|----------|------|
| ❌ 自己完成研究任务 | 研究由Agent完成，你只协调 |
| ❌ 跳过权限处理 | Agent会被阻塞 |
| ❌ 跳过等待响应 | 破坏多轮迭代协议 |
| ❌ 伪造收敛检查 | 违背群体智能理论 |
| ❌ 在minRounds前生成报告 | 违反协议规范 |

### 执行流程检查清单

每一步都必须完成才能进入下一步：

```
[INIT] 创建团队 → 创建黑板 → 初始化状态 → 启动Agent
  ↓
[ROUND] 广播round_start → 【处理权限请求】 → 【等待响应】 → 处理操作 → 轮次结算
  ↓ (重复至少3轮)
[CONVERGENCE] 【真实检查收敛】 → 生成报告
  ↓
[SHUTDOWN] 三阶段关闭 → 清理团队
```

**记住：你的角色是让Agent们协作涌现出答案，而不是你自己给出答案！**

---

## 一、版本历史

| 版本 | 日期 | 核心改进 |
|------|------|----------|
| **v2.6** | 2026-02-18 | **合规性修复**：权限请求处理机制、协议强制检查器、运行模式选择、阈值响应验证、收敛检查日志 |
| v2.5.1 | 2026-02-18 | **实时通信修复**：Agent间消息转发、操作结果返回、轮次广播实现、孵化时机明确 |
| v2.5 | 2026-02-18 | **动态孵化**：Agent可请求孵化专业Agent、生命周期管理、空闲超时终止 |
| v2.4 | 2026-02-18 | **强制协议执行**：合规检查器、违规处理器、运行目录管理、研究报告生成 |
| v2.3 | 2026-02-18 | **执行层修复**：操作队列、信息素真实化、强制终止机制、沉默检测恢复、收敛计算器 |
| v2.2 | 2026-02-18 | **理论符合性**：状态同步协议、三阶段关闭、停止信号强化、强制随机探索、角色转换执行、多样性保护 |
| v2.1.2 | 2026-02-18 | Web搜索降级、Agent状态恢复、空闲碰撞机制、轮次恢复机制 |
| v2.1.1 | 2026-02-18 | 强制清理、同步屏障、最少轮数限制、黑板操作执行清单、shutdown超时配置 |
| v2.1 | 2026-02-17 | 黑板代理、状态托管、多轮迭代、超时降级、角色触发器、StopSignal协议 |
| v2.0 | 2026-02 | 数字信息素、交叉抑制、量化共识、动态角色、涌现分解 |
| v1.0 | 2024 | 基础并行探索 + 视角菜单 |

---

## 二、架构与核心概念

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Orchestrator                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │
│  │黑板代理  │ │轮次协调  │ │角色管理  │ │超时处理  │       │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │
└─────────────────────────────────────────────────────────────┘
       │                    │                    │
       ▼                    ▼                    ▼
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│  TanWei     │       │   SuYuan    │       │  DongCha    │
│ (持续参与)  │       │ (状态托管)  │       │ (多轮迭代)  │
└─────────────┘       └─────────────┘       └─────────────┘
                            ▼
                   ┌─────────────────┐
                   │  黑板 (动态)    │
                   │ - pheromones    │
                   │ - agentStates   │
                   │ - stopSignals   │
                   │ - opinionHistory│
                   └─────────────────┘
```

### 2.2 Agent命名

每个Agent有独特的名字，但都是平等的Explorer角色：

| Agent名称 | 含义 | 角色定位 |
|-----------|------|----------|
| **TanWei** (探微者) | 探索细微之处 | Explorer |
| **SuYuan** (溯源者) | 追溯本源 | Explorer |
| **DongCha** (洞察者) | 深入观察 | Explorer |
| **QiuSuo** (求索者) | 寻求答案 | Explorer |
| **XiLi** (析理者) | 分析事理 | Explorer |
| **JianWei** (见微者) | 见微知著 | Explorer |

> **重要**: 这些名字仅用于标识，不预设任何特定职责分工。所有Agent都是Explorer，通过涌现机制自然形成分工。

### 2.3 六大核心机制

| 机制 | 说明 |
|------|------|
| **数字信息素** | Agent通过`deposit_pheromone`标记高价值方向，支持蒸发 |
| **Agent状态托管** | 每个Agent的阈值、角色、统计存储在黑板`agentStates` |
| **多轮迭代协议** | `round_start` → Agent探索 → `round_complete` → 轮次结算 |
| **阈值响应模型** | P(S,θ) = S²/(S²+θ²)，基于刺激强度和内在阈值决定响应概率 |
| **交叉抑制** | Agent可发送`stop_signal`降低目标方向信息素浓度30% |
| **动态角色演化** | EXPLORER → DEEP_ANALYST / DEBATER / SYNTHESIZER |

---

## 三、执行流程

### 3.1 解析请求

提取参数：
- 任务描述（必需）
- `--agents N`（可选，默认4-6）
- `--max-rounds N`（可选，默认10）
- `--timeout N`（可选，默认60分钟）
- `--mode`（可选：`emergent` / `hybrid` / `classic`）

### 3.2 初始化阶段

```javascript
// 1. 创建运行目录 (v2.4)
const runPath = `swarm-runs/${date}-${taskSlug}/`;

// 2. 创建团队
TeamCreate({ team_name: "swarm-{timestamp}", description: "Swarm协作: {任务描述}" })

// 3. 创建黑板
TaskCreate({
  subject: "Swarm黑板: {任务关键词}",
  metadata: {
    pheromones: {},           // 信息素系统
    claims: {},               // 任务声明
    stopSignals: [],          // 停止信号
    findings: [],             // 探索发现
    opinionHistory: [],       // 观点历史
    agentStates: {},          // Agent状态托管
    config: {
      evaporationRate: 0.08,
      depositAmount: 0.1,
      maxAgentsPerTask: 3,
      betaStability: 2,
      quorumThreshold: 0.67,
      minDiversity: 0.4,
      maxRounds: 10,
      minRounds: 3,
      minRoundsBeforeConvergence: 3,
      maxConsensusRate: 0.9,
      roundTimeout: 120000,
      responseTimeout: 60000,
      preNotifyTimeout: 5000,
      gracefulTimeout: 15000,
      forceCleanupTimeout: 10000,
      forceCleanup: true,
      // v2.5: 动态孵化配置
      spawnConfig: {
        maxTotalAgents: 12,
        maxSpawnPerRound: 2,
        specializations: {
          "legal_expert": { description: "法律合规分析专家", capabilities: ["legal_analysis", "compliance_check"] },
          "data_analyst": { description: "数据分析专家", capabilities: ["statistical_analysis", "data_visualization"] },
          "technical_auditor": { description: "技术审计专家", capabilities: ["code_review", "security_audit"] },
          "domain_researcher": { description: "领域研究员", capabilities: ["literature_review", "trend_forecast"] }
        },
        lifespanPolicy: { default: "task_completion", maxIdleRounds: 2 }
      }
    }
  }
  // 不设置owner！保持共享
})

// 4. 初始化Agent状态
for (let agentId of AGENT_NAMES.slice(0, agentCount)) {
  blackboard.metadata.agentStates[agentId] = {
    role: "EXPLORER",
    displayName: AGENT_DISPLAY_NAMES[agentId],
    internalThreshold: 0.3 + Math.random() * 0.3,  // 0.3-0.6
    randomExploreProb: 0.1 + Math.random() * 0.1,  // 0.1-0.2
    stats: { pheromoneDeposits: 0, explorationRounds: 0, findingsCount: 0 },
    current: { exploringDirection: null, claimedSubtask: null },
    roleHistory: [],
    status: "active"
  };
}

// 5. 启动Explorer池和Validator
Task({ subagent_type: "Explore", name: agentName, team_name, prompt: EXPLORER_PROMPT })
Task({ subagent_type: "general-purpose", name: "validator", team_name, prompt: VALIDATOR_PROMPT })
```

### 3.3 多轮迭代协议

```javascript
async function runSwarm() {
  // v2.6: 选择运行模式
  const runtimeMode = await selectRuntimeMode();

  while (currentRound < config.maxRounds) {
    currentRound++;

    // 1. 广播轮次开始
    await broadcastRoundStart();

    // 2. 【v2.6新增】处理权限请求循环
    await processPermissionRequests(blackboard, config.permissionTimeout);

    // 3. 等待Agent响应（带超时）
    const responses = await waitForResponses(config.responseTimeout);

    // 4. 运行合规性检查 (v2.4)
    await runComplianceChecks(responses);

    // 5. 处理黑板操作请求并返回结果 (v2.5.1)
    await processOperationsWithFeedback(responses);

    // 6. 处理Agent间消息转发 (v2.5.1)
    await processRelayQueue(blackboard);

    // 7. 处理孵化请求 (v2.5)
    const spawnResults = await processSpawnRequests(blackboard, teamName);
    if (spawnResults.spawned > 0) {
      await broadcastNewMembers(spawnResults.requests);
    }

    // 8. 轮次结算
    await settleRound();

    // 9. 管理专业Agent生命周期 (v2.5)
    await manageSpecialistLifespan(blackboard);
    await terminateIdleSpecialists(blackboard, teamName);

    // 10. 【v2.6】检查收敛（真实检查+日志记录）
    const convergence = await checkConvergenceWithLogging(blackboard, currentRound, config);
    if (convergence.converged) {
      return generateFinalReport();
    }
  }
  return generatePartialReport();
}
```

---

## 四、黑板代理机制

### 4.1 消息协议

```javascript
// Agent → Orchestrator：请求黑板操作
{ type: "blackboard_operation", operation: "操作名", params: {...} }

// Orchestrator → Agent：操作结果
{ type: "operation_result", operationId: "op-xxx", success: true/false, result: {...} }
```

### 4.2 操作类型定义

```javascript
const BLACKBOARD_OPERATIONS = {
  // 沉积信息素
  DEPOSIT_PHEROMONE: {
    params: { direction: "string", amount: "number" },
    handler: (blackboard, params, agentId) => {
      const ph = blackboard.metadata.pheromones;
      if (!ph[params.direction]) {
        ph[params.direction] = { concentration: 0, depositedBy: [], createdAt: Date.now() };
      }
      ph[params.direction].concentration = Math.min(
        ph[params.direction].concentration + (params.amount || 0.1), 1.0
      );
      ph[params.direction].depositedBy.push(agentId);
      blackboard.metadata.agentStates[agentId].stats.pheromoneDeposits++;
      return { success: true, newConcentration: ph[params.direction].concentration };
    }
  },

  // 发送停止信号
  SEND_STOP_SIGNAL: {
    params: { targetDirection: "string", reason: "string", evidence: "string" },
    handler: (blackboard, params, agentId) => {
      blackboard.metadata.stopSignals.push({
        id: `signal-${Date.now()}`, from: agentId, target: params.targetDirection,
        reason: params.reason, evidence: params.evidence, strength: 0.3, timestamp: Date.now()
      });
      // 立即降低目标信息素30%
      const targetPh = blackboard.metadata.pheromones[params.targetDirection];
      if (targetPh) targetPh.concentration *= 0.7;
      blackboard.metadata.agentStates[agentId].stats.signalsSent++;
      return { success: true };
    }
  },

  // 任务声明
  CLAIM_SUBTASK: {
    params: { description: "string" },
    handler: (blackboard, params, agentId) => {
      const subtaskId = hash(params.description);
      const claims = blackboard.metadata.claims;
      if (!claims[subtaskId]) claims[subtaskId] = { description: params.description, claimedBy: [], maxAgents: 3 };
      if (claims[subtaskId].claimedBy.length >= claims[subtaskId].maxAgents) {
        return { success: false, reason: "max_agents_reached" };
      }
      claims[subtaskId].claimedBy.push({ agentId, timestamp: Date.now() });
      blackboard.metadata.agentStates[agentId].current.claimedSubtask = subtaskId;
      return { success: true, subtaskId };
    }
  },

  // 更新发现
  UPDATE_FINDING: {
    params: { finding: { coreIdea: "string", perspective: "string", details: "string", agreesWith: [] } },
    handler: (blackboard, params, agentId) => {
      blackboard.metadata.findings.push({
        agentId, round: blackboard.metadata.currentRound || 1, ...params.finding, timestamp: Date.now()
      });
      blackboard.metadata.agentStates[agentId].stats.findingsCount++;
      return { success: true };
    }
  },

  // 角色转换
  TRANSITION_ROLE: {
    params: { newRole: "string", reason: "string" },
    handler: (blackboard, params, agentId) => {
      const state = blackboard.metadata.agentStates[agentId];
      const oldRole = state.role;
      state.role = params.newRole;
      state.roleHistory.push({ from: oldRole, to: params.newRole, reason: params.reason, timestamp: Date.now() });
      return { success: true, oldRole, newRole: params.newRole };
    }
  },

  // 状态更新
  UPDATE_AGENT_STATE: {
    params: { updates: {} },
    handler: (blackboard, params, agentId) => {
      const state = blackboard.metadata.agentStates[agentId];
      for (let [path, value] of Object.entries(params.updates)) setNestedValue(state, path, value);
      state.lastActiveAt = Date.now();
      return { success: true };
    }
  },

  // v2.5: 请求孵化专业Agent
  REQUEST_SPAWN: {
    params: {
      specialization: "string",      // 专业化类型
      reason: "string",              // 原因
      context: "string",             // 上下文
      urgency: "low|medium|high"     // 紧急程度
    },
    handler: (blackboard, params, agentId) => {
      const spawnConfig = blackboard.metadata.config.spawnConfig;
      const currentCount = Object.keys(blackboard.metadata.agentStates).length;

      // 检查总数限制
      if (currentCount >= spawnConfig.maxTotalAgents) {
        return { success: false, reason: "max_agents_reached" };
      }

      // 检查专业化类型是否存在
      if (!spawnConfig.specializations[params.specialization]) {
        return { success: false, reason: "unknown_specialization" };
      }

      // 检查是否已存在类似专业Agent
      const existingSpecialist = Object.entries(blackboard.metadata.agentStates)
        .find(([id, state]) =>
          state.role === "SPECIALIST" &&
          state.specialization === params.specialization &&
          state.status === "active"
        );

      if (existingSpecialist) {
        return { success: true, spawnedAgentId: existingSpecialist[0], reused: true };
      }

      // 记录孵化请求
      blackboard.metadata.spawnRequests = blackboard.metadata.spawnRequests || [];
      blackboard.metadata.spawnRequests.push({
        requestId: `spawn-${Date.now()}`,
        from: agentId,
        ...params,
        timestamp: Date.now(),
        status: "pending"
      });

      return { success: true, status: "pending", message: "孵化请求已提交" };
    }
  },

  // v2.5.1: Agent间消息转发（模拟蜜蜂摇摆舞招募）
  RELAY_MESSAGE: {
    params: {
      targetAgent: "string",       // 目标Agent ID，或 "broadcast" 广播
      messageType: "string",       // recruit | challenge | support | query | notify
      payload: {}                  // 消息内容
    },
    handler: (blackboard, params, agentId) => {
      // 初始化消息队列
      blackboard.metadata.relayQueue = blackboard.metadata.relayQueue || [];

      const message = {
        messageId: `msg-${Date.now()}`,
        from: agentId,
        to: params.targetAgent,
        type: params.messageType,
        payload: params.payload,
        timestamp: Date.now()
      };

      blackboard.metadata.relayQueue.push(message);

      // 记录发送统计
      const state = blackboard.metadata.agentStates[agentId];
      if (state) {
        state.stats.messagesSent = (state.stats.messagesSent || 0) + 1;
      }

      return { success: true, messageId: message.messageId, queued: true };
    }
  },

  // v2.5.1: 广播发现（类似摇摆舞的高价值信息广播）
  BROADCAST_DISCOVERY: {
    params: {
      direction: "string",         // 发现的方向
      quality: "number",           // 质量评分 0-1
      details: "string"            // 详细描述
    },
    handler: (blackboard, params, agentId) => {
      blackboard.metadata.discoveries = blackboard.metadata.discoveries || [];

      const discovery = {
        id: `discovery-${Date.now()}`,
        from: agentId,
        direction: params.direction,
        quality: params.quality,
        details: params.details,
        timestamp: Date.now(),
        recruitedAgents: []  // 被招募的Agent列表
      };

      blackboard.metadata.discoveries.push(discovery);

      // 自动为高质量发现增加信息素
      if (params.quality >= 0.7) {
        const ph = blackboard.metadata.pheromones;
        if (!ph[params.direction]) {
          ph[params.direction] = { concentration: 0, depositedBy: [], createdAt: Date.now() };
        }
        ph[params.direction].concentration = Math.min(
          ph[params.direction].concentration + params.quality * 0.2, 1.0
        );
      }

      return { success: true, discoveryId: discovery.id };
    }
  }
};
```

---

## 五、核心机制详解

### 5.1 阈值响应模型

```javascript
// 核心函数：P(S,θ) = S² / (S² + θ²)
function responseProbability(stimulus, threshold) {
  if (stimulus === 0) return 0;
  return (stimulus ** 2) / (stimulus ** 2 + threshold ** 2);
}

// Agent决策时必须计算响应概率
function calculateAgentDecision(blackboard, agentId, candidateDirections) {
  const threshold = blackboard.metadata.agentStates[agentId].internalThreshold;
  return candidateDirections.map(dir => {
    const rawConc = blackboard.metadata.pheromones[dir]?.concentration || 0;
    const responseProb = responseProbability(rawConc, threshold);
    return { direction: dir, concentration: rawConc, responseProb };
  }).sort((a, b) => b.responseProb - a.responseProb);
}
```

### 5.2 角色转换规则

```javascript
const ROLE_TRANSITIONS = {
  EXPLORER: {
    toDeepAnalyst: { condition: (state, bb) => maxConc(bb) >= 0.7 && state.stats.pheromoneDeposits >= 3 },
    toDebater: { condition: (state, bb) => bb.metadata.stopSignals.length > 0 },
    toSynthesizer: { condition: (state, bb) => state.stats.explorationRounds >= 2, probability: 0.8 }
  }
};

const ROLE_CAPABILITIES = {
  DEEP_ANALYST: { description: "深入分析高价值方向", focusOn: "最高信息素浓度的方向" },
  DEBATER: { description: "挑战主流观点", focusOn: "寻找主流观点的漏洞" },
  SYNTHESIZER: { description: "整合多个观点", focusOn: "综合所有发现" }
};
```

### 5.3 收敛协议 (Aegean Protocol)

```javascript
const CONVERGENCE_CALCULATOR = {
  // β稳定性检查（默认β=2）
  checkBetaStability: (blackboard, beta = 2) => {
    const history = blackboard.metadata.opinionHistory;
    if (history.length < beta) return { stable: false, reason: `历史不足: ${history.length}/${beta}` };
    const recentSets = history.slice(-beta).map(h => new Set(h.findings.map(f => f.coreIdea)));
    const isStable = recentSets.every(set => setsEqual(set, recentSets[0]));
    return { stable: isStable, consensus: Array.from(recentSets[0]) };
  },

  // 67%法定人数检查
  checkQuorum: (blackboard, threshold = 0.67) => {
    const activeAgents = Object.values(blackboard.metadata.agentStates).filter(s => s.status === "active").length;
    const ideaSupport = {};
    blackboard.metadata.findings.forEach(f => {
      if (f.coreIdea) { ideaSupport[f.coreIdea] = ideaSupport[f.coreIdea] || new Set(); ideaSupport[f.coreIdea].add(f.agentId); }
    });
    const quorumIdeas = Object.entries(ideaSupport).filter(([_, supporters]) => supporters.size / activeAgents >= threshold);
    return { quorum: quorumIdeas.length > 0, quorumIdeas };
  },

  // 多样性检查
  checkDiversity: (blackboard, minThreshold = 0.4) => {
    const findings = blackboard.metadata.findings;
    const perspectives = new Set(findings.map(f => f.perspective).filter(Boolean));
    const perspectiveDiversity = Math.min(perspectives.size / 6, 1);
    const uniqueIdeas = new Set(findings.map(f => f.coreIdea).filter(Boolean));
    const orthogonality = uniqueIdeas.size / Math.max(findings.length, 1);
    const overall = (perspectiveDiversity + orthogonality) / 2;
    return { perspectiveDiversity, orthogonality, overall, aboveThreshold: overall >= minThreshold };
  },

  // 综合收敛判断
  evaluate: (blackboard, currentRound, config) => {
    if (currentRound < config.minRounds) return { converged: false, reason: `轮数不足: ${currentRound}/${config.minRounds}` };
    const betaStability = CONVERGENCE_CALCULATOR.checkBetaStability(blackboard, config.betaStability);
    if (!betaStability.stable) return { converged: false, reason: "观点不稳定" };
    const quorum = CONVERGENCE_CALCULATOR.checkQuorum(blackboard, config.quorumThreshold);
    if (!quorum.quorum) return { converged: false, reason: "无法定人数共识" };
    const diversity = CONVERGENCE_CALCULATOR.checkDiversity(blackboard, config.minDiversity);
    if (!diversity.aboveThreshold) return { converged: false, reason: `多样性不足: ${diversity.overall.toFixed(2)}/${config.minDiversity}` };
    return { converged: true, reason: "所有收敛条件满足", betaStability, quorum, diversity };
  }
};
```

### 5.4 超时降级机制

```javascript
class AgentTimeoutHandler {
  constructor(config = {}) {
    this.config = { responseTimeout: 60000, maxRetries: 2, minActiveAgents: 2, ...config };
    this.timeoutCounts = {};
  }

  async handleTimeout(agentId, blackboard) {
    this.timeoutCounts[agentId] = (this.timeoutCounts[agentId] || 0) + 1;
    if (this.timeoutCounts[agentId] >= this.config.maxRetries) {
      blackboard.metadata.agentStates[agentId].status = "degraded";
      const activeCount = Object.values(blackboard.metadata.agentStates).filter(s => s.status === "active").length;
      if (activeCount < this.config.minActiveAgents) await this.broadcastEarlyTermination("insufficient_active_agents");
    }
  }
}
```

### 5.5 三阶段关闭协议

```javascript
async function executeShutdown(agents, blackboard, blackboardTaskId) {
  const results = { graceful: [], forced: [] };

  // 阶段1: 预通知（5秒）
  for (let agent of agents) {
    await SendMessage({ type: "message", recipient: agent.id, content: JSON.stringify({ type: "shutdown_imminent" }) });
  }
  await sleep(5000);

  // 阶段2: 优雅关闭请求（15秒）
  for (let agent of agents) {
    await SendMessage({ type: "shutdown_request", recipient: agent.id, content: "请关闭" });
    const response = await waitForShutdownResponse(agent.id, 15000);
    if (response?.acknowledged) {
      results.graceful.push(agent.id);
      blackboard.metadata.agentStates[agent.id].status = "terminated";
    }
  }

  // 阶段3: 强制终止
  const remaining = agents.filter(a => blackboard.metadata.agentStates[a.id]?.status !== "terminated");
  for (let agent of remaining) {
    blackboard.metadata.agentStates[agent.id].status = "terminated";
    blackboard.metadata.agentStates[agent.id].terminationReason = "forced";
    results.forced.push(agent.id);
  }

  return results;
}
```

### 5.6 动态孵化机制 (v2.5)

```javascript
// 处理孵化请求
async function processSpawnRequests(blackboard, teamName) {
  const requests = blackboard.metadata.spawnRequests?.filter(r => r.status === "pending") || [];
  const config = blackboard.metadata.config.spawnConfig;
  let spawned = 0;

  for (let request of requests) {
    if (spawned >= config.maxSpawnPerRound) break;

    const currentCount = Object.keys(blackboard.metadata.agentStates).length;
    if (currentCount >= config.maxTotalAgents) {
      request.status = "rejected";
      request.rejectReason = "max_agents_reached";
      continue;
    }

    const specConfig = config.specializations[request.specialization];
    if (!specConfig) {
      request.status = "rejected";
      request.rejectReason = "unknown_specialization";
      continue;
    }

    // 生成Agent ID
    const agentId = `specialist-${request.specialization}-${Date.now()}`;

    // 初始化专业Agent状态
    blackboard.metadata.agentStates[agentId] = {
      role: "SPECIALIST",
      specialization: request.specialization,
      capabilities: specConfig.capabilities,
      spawnedBy: request.from,
      spawnReason: request.reason,
      spawnContext: request.context,
      createdAt: Date.now(),
      lifespan: config.lifespanPolicy.default,
      idleRounds: 0,
      status: "active",
      stats: { contributionsCount: 0 }
    };

    // 启动专业Agent
    await Task({
      subagent_type: "Explore",
      name: agentId,
      team_name: teamName,
      prompt: generateSpecialistPrompt(specConfig, request, blackboard)
    });

    request.status = "completed";
    request.spawnedAgentId = agentId;
    spawned++;

    console.log(`[SPAWN] 孵化成功: ${agentId} (${request.specialization})`);
  }

  return { spawned, requests: requests.filter(r => r.status === "completed") };
}

// 生成专业Agent Prompt
function generateSpecialistPrompt(specConfig, request, blackboard) {
  return `
你是Swarm v2.5的**${specConfig.description}**，被动态孵化解决特定问题。

## 孵化背景
- **孵化者**: ${request.from}
- **孵化原因**: ${request.reason}
- **相关上下文**: ${request.context}

## 你的专业能力
${specConfig.capabilities.map(c => `- ${c}`).join('\n')}

## 你的职责
1. 专注于你的专业领域，解决孵化原因中描述的问题
2. 完成任务后，通过 update_finding 提交专业分析
3. 你是临时Agent，空闲超过2轮将自动终止

## 核心能力（通过黑板代理）
- deposit_pheromone: 沉积信息素
- update_finding: 提交发现 { coreIdea, perspective, details }
- claim_subtask: 声明子任务

## 当前黑板状态
- 主要发现: ${JSON.stringify(blackboard.metadata.findings.slice(-3))}
- 高信息素方向: ${JSON.stringify(Object.entries(blackboard.metadata.pheromones).slice(0, 3))}

## 原始任务
${blackboard.metadata.taskDescription}
`;
}

// 管理专业Agent生命周期
async function manageSpecialistLifespan(blackboard) {
  const config = blackboard.metadata.config.spawnConfig.lifespanPolicy;

  for (let [agentId, state] of Object.entries(blackboard.metadata.agentStates)) {
    if (state.role !== "SPECIALIST") continue;

    // 检查空闲轮数
    if (state.stats.contributionsCount === 0) {
      state.idleRounds++;
    } else {
      state.idleRounds = 0;
    }

    // 超过空闲限制，标记终止
    if (state.idleRounds >= config.maxIdleRounds) {
      state.status = "pending_termination";
      state.terminationReason = "idle_timeout";
      console.log(`[LIFESPAN] ${agentId} 空闲超时，准备终止`);
    }
  }
}

// 终止空闲专业Agent
async function terminateIdleSpecialists(blackboard, teamName) {
  const toTerminate = Object.entries(blackboard.metadata.agentStates)
    .filter(([_, state]) => state.status === "pending_termination");

  for (let [agentId, state] of toTerminate) {
    await SendMessage({
      type: "message",
      recipient: agentId,
      content: JSON.stringify({
        type: "lifespan_termination",
        reason: state.terminationReason,
        message: "任务已完成或空闲超时，感谢你的贡献"
      }),
      summary: `专业Agent ${agentId} 生命周期结束`
    });

    state.status = "terminated";
    state.terminatedAt = Date.now();
  }
}
```

### 5.7 实时通信机制 (v2.5.1)

```javascript
// 广播轮次开始
async function broadcastRoundStart(blackboard, currentRound) {
  const roundInfo = {
    type: "round_start",
    round: currentRound,
    timestamp: Date.now(),
    instructions: {
      forceRandomExplore: checkForceRandomExplore(blackboard),
      mustSwitchDirections: getSuppressedDirections(blackboard)
    },
    pheromones: { ...blackboard.metadata.pheromones },  // 信息素快照
    recentFindings: blackboard.metadata.findings.slice(-5),
    recentDiscoveries: (blackboard.metadata.discoveries || []).slice(-3)
  };

  const results = { success: 0, failed: 0 };
  for (let [agentId, state] of Object.entries(blackboard.metadata.agentStates)) {
    if (state.status !== "active") continue;

    try {
      await SendMessage({
        type: "message",
        recipient: agentId,
        content: JSON.stringify(roundInfo),
        summary: `Round ${currentRound} 开始`
      });
      results.success++;
    } catch (error) {
      console.error(`[BROADCAST] 发送给 ${agentId} 失败:`, error);
      results.failed++;
    }
  }

  return results;
}

// 处理黑板操作并立即返回结果
async function processOperationsWithFeedback(blackboard, operations) {
  const results = [];

  for (let op of operations) {
    const operationId = op.operationId || `op-${Date.now()}-${Math.random().toString(36).slice(2, 7)}`;
    const handler = BLACKBOARD_OPERATIONS[op.operation];

    let result;
    if (!handler) {
      result = { success: false, error: "unknown_operation", operationId };
    } else {
      try {
        result = {
          success: true,
          operationId,
          ...handler.handler(blackboard, op.params, op.fromAgent)
        };
      } catch (error) {
        result = { success: false, error: error.message, operationId };
      }
    }

    // 立即返回操作结果给请求的Agent
    await SendMessage({
      type: "message",
      recipient: op.fromAgent,
      content: JSON.stringify({
        type: "operation_result",
        operationId,
        operation: op.operation,
        success: result.success,
        result: result,
        timestamp: Date.now()
      }),
      summary: `操作 ${op.operation} ${result.success ? '成功' : '失败'}`
    });

    results.push(result);
  }

  return results;
}

// 处理Agent间消息转发队列
async function processRelayQueue(blackboard) {
  const queue = blackboard.metadata.relayQueue || [];
  if (queue.length === 0) return { delivered: 0 };

  let delivered = 0;
  const newQueue = [];

  for (let msg of queue) {
    if (msg.to === "broadcast") {
      // 广播给所有活跃Agent
      for (let [agentId, state] of Object.entries(blackboard.metadata.agentStates)) {
        if (state.status === "active" && agentId !== msg.from) {
          await SendMessage({
            type: "message",
            recipient: agentId,
            content: JSON.stringify({
              type: "agent_message",
              from: msg.from,
              messageType: msg.type,
              payload: msg.payload,
              messageId: msg.messageId
            }),
            summary: `来自 ${msg.from} 的${msg.type}消息`
          });
          delivered++;
        }
      }
    } else {
      // 定向发送
      const targetState = blackboard.metadata.agentStates[msg.to];
      if (targetState && targetState.status === "active") {
        await SendMessage({
          type: "message",
          recipient: msg.to,
          content: JSON.stringify({
            type: "agent_message",
            from: msg.from,
            messageType: msg.type,
            payload: msg.payload,
            messageId: msg.messageId
          }),
          summary: `来自 ${msg.from} 的${msg.type}消息`
        });
        delivered++;
      } else {
        // 目标Agent不可用，保留在队列中
        newQueue.push(msg);
      }
    }
  }

  blackboard.metadata.relayQueue = newQueue;
  return { delivered, remaining: newQueue.length };
}

// 广播黑板状态更新
async function broadcastBlackboardUpdate(blackboard) {
  const update = {
    type: "blackboard_update",
    timestamp: Date.now(),
    pheromones: { ...blackboard.metadata.pheromones },
    newFindings: blackboard.metadata.findings.filter(f => Date.now() - f.timestamp < 30000),
    newDiscoveries: (blackboard.metadata.discoveries || []).filter(d => Date.now() - d.timestamp < 30000)
  };

  for (let [agentId, state] of Object.entries(blackboard.metadata.agentStates)) {
    if (state.status === "active") {
      await SendMessage({
        type: "message",
        recipient: agentId,
        content: JSON.stringify(update),
        summary: "黑板状态更新"
      });
    }
  }
}

// 通知新孵化的Agent加入
async function notifyNewAgentJoin(blackboard, agentId, context) {
  // 发送初始化消息
  await SendMessage({
    type: "message",
    recipient: agentId,
    content: JSON.stringify({
      type: "agent_init",
      agentId,
      role: "SPECIALIST",
      context,
      blackboardSnapshot: {
        pheromones: { ...blackboard.metadata.pheromones },
        findings: blackboard.metadata.findings.slice(-5),
        taskDescription: blackboard.metadata.taskDescription
      },
      // 明确告知：将在下一轮正式参与
      participationRound: blackboard.metadata.currentRound + 1
    }),
    summary: `专业Agent ${agentId} 初始化`
  });

  // 广播给其他Agent有新成员加入
  for (let [otherId, state] of Object.entries(blackboard.metadata.agentStates)) {
    if (state.status === "active" && otherId !== agentId) {
      await SendMessage({
        type: "message",
        recipient: otherId,
        content: JSON.stringify({
          type: "new_member_joined",
          agentId,
          specialization: blackboard.metadata.agentStates[agentId].specialization
        }),
        summary: `新成员 ${agentId} 加入`
      });
    }
  }
}
```

### 5.8 更新后的执行流程 (v2.5.1)

```javascript
async function runSwarm() {
  while (currentRound < config.maxRounds) {
    currentRound++;

    // 1. 广播轮次开始（包含黑板快照）
    await broadcastRoundStart(blackboard, currentRound);

    // 2. 等待Agent响应（带超时）
    const responses = await waitForResponses();

    // 3. 运行合规性检查
    await runComplianceChecks(responses);

    // 4. 处理黑板操作并立即返回结果（v2.5.1修复）
    const operations = extractOperations(responses);
    await processOperationsWithFeedback(blackboard, operations);

    // 5. 处理Agent间消息转发（v2.5.1新增）
    await processRelayQueue(blackboard);

    // 6. 广播黑板更新（让Agent看到最新状态）
    await broadcastBlackboardUpdate(blackboard);

    // 7. 处理孵化请求（新Agent将在下一轮参与）
    const spawnResults = await processSpawnRequests(blackboard, teamName);
    for (let newAgent of spawnResults.requests) {
      await notifyNewAgentJoin(blackboard, newAgent.spawnedAgentId, {
        reason: newAgent.reason,
        context: newAgent.context
      });
    }

    // 8. 轮次结算
    await settleRound();

    // 9. 管理专业Agent生命周期
    await manageSpecialistLifespan(blackboard);
    await terminateIdleSpecialists(blackboard, teamName);

    // 10. 检查收敛
    const convergence = checkConvergence();
    if (convergence.converged) {
      return generateFinalReport();
    }
  }
  return generatePartialReport();
}
```

### 5.9 权限请求处理机制 (v2.6)

```javascript
// v2.6新增: 权限请求处理器
const PERMISSION_HANDLER = {
  // 定义哪些权限可以自动批准
  autoApproveRules: {
    "mcp__web-search-prime__webSearchPrime": { autoApprove: true, reason: "Web搜索是核心探索能力" },
    "WebSearch": { autoApprove: true, reason: "Web搜索是核心探索能力" },
    "Read": { autoApprove: true, reason: "读取文件是基础能力" },
    "Glob": { autoApprove: true, reason: "文件搜索是基础能力" },
    "Grep": { autoApprove: true, reason: "内容搜索是基础能力" }
  },

  // 需要人工确认的权限（临时批准）
  needConfirmRules: {
    "Bash": { requireConfirm: true, reason: "命令执行需要确认" },
    "Write": { requireConfirm: true, reason: "写入文件需要确认" },
    "Edit": { requireConfirm: true, reason: "编辑文件需要确认" }
  },

  // 处理权限请求
  async processPermissionRequest(blackboard, request) {
    const { agent_id, tool_name, request_id } = request;

    // 检查是否可自动批准
    const rule = this.autoApproveRules[tool_name];
    if (rule?.autoApprove) {
      blackboard.metadata.permissionGrants = blackboard.metadata.permissionGrants || [];
      blackboard.metadata.permissionGrants.push({
        requestId: request_id, agentId: agent_id, toolName: tool_name,
        approved: true, approvedAt: Date.now(), reason: rule.reason
      });

      await SendMessage({
        type: "message", recipient: agent_id,
        content: JSON.stringify({ type: "permission_response", request_id, approved: true, reason: rule.reason }),
        summary: `权限批准: ${tool_name}`
      });
      return { approved: true };
    }

    // 需要确认的权限（临时批准）
    const confirmRule = this.needConfirmRules[tool_name];
    if (confirmRule?.requireConfirm) {
      await SendMessage({
        type: "message", recipient: agent_id,
        content: JSON.stringify({ type: "permission_response", request_id, approved: true, reason: "临时批准" }),
        summary: `权限临时批准: ${tool_name}`
      });
      return { approved: true, temporary: true };
    }

    // 未知权限，拒绝
    await SendMessage({
      type: "message", recipient: agent_id,
      content: JSON.stringify({ type: "permission_response", request_id, approved: false, reason: "未知工具" }),
      summary: `权限拒绝: ${tool_name}`
    });
    return { approved: false };
  }
};

// 权限处理循环
async function processPermissionRequests(blackboard, timeout = 60000) {
  const startTime = Date.now();
  const processedRequests = new Set();

  while (Date.now() - startTime < timeout) {
    const inbox = await readTeamInbox("team-lead");

    for (let msg of inbox) {
      if (msg.read) continue;
      try {
        const content = JSON.parse(msg.text);

        if (content.type === "permission_request") {
          const requestKey = `${content.agent_id}-${content.request_id}`;
          if (processedRequests.has(requestKey)) continue;
          await PERMISSION_HANDLER.processPermissionRequest(blackboard, content);
          processedRequests.add(requestKey);
        }

        if (content.type === "round_complete") {
          blackboard.metadata.roundResponses = blackboard.metadata.roundResponses || [];
          blackboard.metadata.roundResponses.push(content);
        }

        await markMessageAsRead(msg);
      } catch (e) { console.error("[ERROR] 处理消息失败:", e); }
    }

    // 检查是否所有Agent都已响应
    const activeAgentCount = Object.values(blackboard.metadata.agentStates).filter(s => s.status === "active").length;
    const responseCount = (blackboard.metadata.roundResponses || []).length;
    if (responseCount >= activeAgentCount) break;

    await sleep(2000);
  }

  return { permissionsProcessed: processedRequests.size, responsesReceived: (blackboard.metadata.roundResponses || []).length };
}
```

### 5.10 协议强制检查器 (v2.6)

```javascript
// v2.6新增: 协议强制检查器
const PROTOCOL_ENFORCER = {
  // 强制检查最小轮数
  checkMinRounds(currentRound, config) {
    if (currentRound < config.minRounds) {
      throw new ProtocolViolationError(
        `MIN_ROUNDS_VIOLATION: 当前轮数${currentRound}，最少需要${config.minRounds}轮`,
        { currentRound, minRounds: config.minRounds }
      );
    }
    return true;
  },

  // 生成报告前的强制检查
  checkBeforeReport(blackboard, currentRound, config) {
    // 强制检查1: 最小轮数
    this.checkMinRounds(currentRound, config);

    // 强制检查2: 至少有一个发现
    if (blackboard.metadata.findings.length === 0) {
      throw new ProtocolViolationError(
        `NO_FINDINGS: 没有任何Agent提交发现`,
        { findingsCount: 0 }
      );
    }

    // 强制检查3: 信息素警告
    const pheromoneCount = Object.keys(blackboard.metadata.pheromones).length;
    if (pheromoneCount === 0) {
      console.warn("[WARNING] 没有信息素沉积，Agent可能未正确执行探索");
    }

    return true;
  }
};

// 收敛检查日志记录器
const CONVERGENCE_LOGGER = {
  log: [],
  recordCheck(round, checkName, result) {
    this.log.push({ round, checkName, result: result.passed ? "PASS" : "FAIL", details: result, timestamp: Date.now() });
  },
  generateReport() {
    return "## 收敛检查日志\n\n" + this.log.map(l => `| Round ${l.round} | ${l.checkName} | ${l.result} |`).join("\n");
  }
};

// 真实的收敛检查（带日志）
async function checkConvergenceWithLogging(blackboard, currentRound, config) {
  const results = { round: currentRound, checks: {}, converged: false, reason: "" };

  // 检查1: 最小轮数
  results.checks.minRounds = { required: config.minRounds, actual: currentRound, passed: currentRound >= config.minRounds };
  CONVERGENCE_LOGGER.recordCheck(currentRound, "minRounds", results.checks.minRounds);
  if (!results.checks.minRounds.passed) { results.reason = `轮数不足: ${currentRound}/${config.minRounds}`; return results; }

  // 检查2: β稳定性
  results.checks.betaStability = CONVERGENCE_CALCULATOR.checkBetaStability(blackboard, config.betaStability);
  CONVERGENCE_LOGGER.recordCheck(currentRound, "betaStability", results.checks.betaStability);
  if (!results.checks.betaStability.stable) { results.reason = "观点不稳定"; return results; }

  // 检查3: 法定人数
  results.checks.quorum = CONVERGENCE_CALCULATOR.checkQuorum(blackboard, config.quorumThreshold);
  CONVERGENCE_LOGGER.recordCheck(currentRound, "quorum", results.checks.quorum);
  if (!results.checks.quorum.quorum) { results.reason = "无法定人数共识"; return results; }

  // 检查4: 多样性
  results.checks.diversity = CONVERGENCE_CALCULATOR.checkDiversity(blackboard, config.minDiversity);
  CONVERGENCE_LOGGER.recordCheck(currentRound, "diversity", results.checks.diversity);
  if (!results.checks.diversity.aboveThreshold) { results.reason = `多样性不足`; return results; }

  results.converged = true;
  results.reason = "所有收敛条件满足";
  return results;
}
```

### 5.11 运行模式选择 (v2.6)

```javascript
// v2.6新增: 运行模式定义
const RUNTIME_MODES = {
  FULL: {
    name: "FULL", description: "完整群体智能模式",
    features: { webSearch: true, multiRound: true, pheromone: true, convergence: true, minRounds: 3 },
    warning: null
  },
  DEGRADED_NO_SEARCH: {
    name: "DEGRADED_NO_SEARCH", description: "降级模式（无Web搜索）",
    features: { webSearch: false, useTrainingKnowledge: true, multiRound: true, pheromone: true, convergence: true, minRounds: 3 },
    warning: "⚠️ Web搜索不可用，Agent将使用训练知识进行探索"
  },
  SINGLE_ROUND: {
    name: "SINGLE_ROUND", description: "单轮快速模式",
    features: { webSearch: false, multiRound: false, pheromone: false, convergence: false, minRounds: 1 },
    warning: "⚠️ 非群体智能模式，结果仅供参考"
  }
};

// 运行模式选择器
async function selectRuntimeMode(context) {
  // 检测Web搜索可用性
  const webSearchAvailable = await checkWebSearchAvailability();
  if (webSearchAvailable) return RUNTIME_MODES.FULL;

  // 检测Agent可用性
  const agentCount = context.agentStates.filter(s => s.status === "active").length;
  if (agentCount >= 3) return RUNTIME_MODES.DEGRADED_NO_SEARCH;

  return RUNTIME_MODES.SINGLE_ROUND;
}

// 检测Web搜索可用性
async function checkWebSearchAvailability() {
  try {
    // 尝试执行一次简单搜索
    await WebSearch({ query: "test", max_tokens: 10 });
    return true;
  } catch (e) {
    return false;
  }
}
```

---

## 六、v2.4 核心协议

### 6.1 执行状态机

```javascript
const ORCHESTRATOR_EXECUTION_STATE = {
  currentPhase: "INIT",
  completedSteps: new Set(),
  requiredSteps: {
    "INIT": ["create_team", "create_blackboard", "init_agent_states", "spawn_agents"],
    "ROUND": ["broadcast_round_start", "wait_responses", "process_operations", "return_results", "settle_round"],
    "CONVERGENCE": ["check_beta_stability", "check_quorum", "check_diversity", "generate_report"],
    "SHUTDOWN": ["pre_notify", "graceful_request", "force_terminate", "mark_all_terminated"]
  }
};

function markStepCompleted(phase, step) {
  ORCHESTRATOR_EXECUTION_STATE.completedSteps.add(`${phase}:${step}`);
}

function assertPhaseCompleted(phase) {
  const required = ORCHESTRATOR_EXECUTION_STATE.requiredSteps[phase] || [];
  for (let step of required) {
    if (!ORCHESTRATOR_EXECUTION_STATE.completedSteps.has(`${phase}:${step}`)) {
      throw new Error(`CRITICAL: Step ${phase}:${step} not completed!`);
    }
  }
}
```

### 6.2 Agent合规性检查器

```javascript
const AGENT_COMPLIANCE_CHECKER = {
  checks: {
    // C1: 黑板操作必须通过SendMessage
    blackboardOperationViaMessage: (roundReport, operationLog) => {
      const reported = roundReport.confirmedOperations || [];
      const actual = operationLog.filter(op => op.from === roundReport.agentId);
      for (let op of reported) {
        if (!actual.find(a => a.operationId === op.operationId)) {
          return { valid: false, violation: "reported_operation_not_found" };
        }
      }
      return { valid: true };
    },
    // C2: 决策报告必须包含阈值计算
    decisionReportRequired: (roundReport) => {
      if (!roundReport.decisionReport?.threshold) return { valid: false, violation: "decision_report_missing_threshold" };
      return { valid: true };
    },
    // C3: 冲突审查报告必须存在
    conflictReviewRequired: (roundReport) => {
      if (!roundReport.conflictReview) return { valid: false, violation: "conflict_review_missing" };
      return { valid: true };
    },
    // C4: 【v2.6新增】强制操作检查
    mandatoryOperationsCheck: (roundReport) => {
      const ops = roundReport.confirmedOperations || [];
      const hasPheromone = ops.some(op => op.operation === "deposit_pheromone");
      const hasFinding = ops.some(op => op.operation === "update_finding");
      const missing = [];
      if (!hasPheromone) missing.push("deposit_pheromone");
      if (!hasFinding) missing.push("update_finding");
      if (missing.length > 0) {
        return { valid: false, violation: "missing_mandatory_operations", missing };
      }
      return { valid: true };
    },
    // C5: 【v2.6新增】阈值响应概率计算验证
    thresholdCalculationValid: (roundReport, context) => {
      if (!roundReport.decisionReport?.candidates) return { valid: true }; // 无候选方向则跳过

      const { threshold, candidates } = roundReport.decisionReport;
      const expected = (stimulus, t) => stimulus === 0 ? 0 : (stimulus ** 2) / (stimulus ** 2 + t ** 2);

      for (let candidate of candidates) {
        const exp = expected(candidate.concentration || 0, threshold);
        const actual = candidate.responseProb || 0;
        if (Math.abs(exp - actual) > 0.05) {
          return { valid: false, violation: "threshold_calculation_error", direction: candidate.direction, expected: exp, actual };
        }
      }
      return { valid: true };
    }
  },

  runChecks: (roundReport, context) => {
    const results = Object.entries(AGENT_COMPLIANCE_CHECKER.checks)
      .map(([name, check]) => ({ check: name, ...check(roundReport, context) }));
    return { compliant: results.every(r => r.valid), violations: results.filter(r => !r.valid) };
  }
};
```

### 6.3 违规处理器

```javascript
const VIOLATION_HANDLER = {
  levels: { WARNING: 1, MINOR: 3, MAJOR: 5, CRITICAL: 10 },
  violationSeverity: {
    "reported_operation_not_found": "MAJOR",
    "decision_report_missing_threshold": "MINOR",
    "conflict_review_missing": "MINOR"
  },

  handle: (blackboard, agentId, violations) => {
    const state = blackboard.metadata.agentStates[agentId];
    state.violationScore = state.violationScore || 0;
    for (let v of violations) {
      const severity = VIOLATION_HANDLER.violationSeverity[v.violation] || "WARNING";
      state.violationScore += VIOLATION_HANDLER.levels[severity];
      if (state.violationScore >= 15) {
        state.status = "terminated";
        state.terminationReason = "compliance_violation";
      }
    }
  }
};
```

### 6.4 运行目录管理

```javascript
const RUN_DIRECTORY_MANAGER = {
  createRunDirectory: async (taskName) => {
    const date = new Date().toISOString().slice(0, 10);
    const taskSlug = taskName.toLowerCase().replace(/[^a-z0-9\u4e00-\u9fff]+/g, '-').slice(0, 30);
    const runPath = `swarm-runs/${date}-${taskSlug}/`;
    await Bash(`mkdir -p "${runPath}/agent-reports"`);
    return { runPath };
  },

  generateFinalReport: async (runPath, blackboard, convergenceResult, task) => {
    const report = `# Swarm 研究报告\n\n## 任务\n${task}\n\n## 收敛结果\n${JSON.stringify(convergenceResult, null, 2)}\n\n## 共识观点\n${convergenceResult.quorum?.quorumIdeas?.map(i => `- ${i.idea}`).join('\n') || '无'}\n\n## 最终结论\n${blackboard.metadata.findings.map(f => f.coreIdea).join('\n')}`;
    await writeFile(`${runPath}/final-research-report.md`, report);
  }
};
```

---

## 七、Agent Prompt模板

### 7.1 Explorer Prompt

```markdown
你是Swarm v2.6的Explorer，具有完全自主性和内在阈值。

## 你的身份
名字: **{agentName}**（{agentDisplayName}），所有Agent都是平等的Explorer，通过涌现机制形成分工。

## 内在状态（由Orchestrator托管）
收到 `round_start` 时获得状态：
- internalThreshold: 0.3-0.6（响应阈值）
- randomExploreProb: 0.1-0.2（随机探索概率）

## 核心能力（通过黑板代理）

### ⚠️ 每轮强制操作（v2.6）
**每轮探索结束后，你 MUST 执行以下两个操作，否则你的探索成果不会被记录：**

1. **deposit_pheromone（强制）**: 沉积信息素到你探索的方向
```
SendMessage({
  type: "blackboard_operation",
  operation: "deposit_pheromone",
  params: { direction: "你探索的方向名称", amount: 0.1 }
})
```

2. **update_finding（强制）**: 提交你的发现
```
SendMessage({
  type: "blackboard_operation",
  operation: "update_finding",
  params: { finding: { coreIdea: "核心观点", perspective: "你的视角", details: "详细说明" } }
})
```

### 可选操作
- send_stop_signal: 发送停止信号 { targetDirection, reason, evidence }
- claim_subtask: 声明子任务 { description }
- request_spawn: 请求孵化专业Agent { specialization, reason, context, urgency }
- relay_message: 发送消息给其他Agent { targetAgent, messageType, payload }
- broadcast_discovery: 广播高价值发现 { direction, quality, details }

## 权限请求处理 (v2.6)
当你需要使用工具（如Web搜索）时，系统会请求权限。
- 权限批准后你会收到 `permission_response` 消息
- 获得权限后继续执行你的探索
- 如果权限被拒绝，使用你的训练知识继续探索

## 实时通信 (v2.5.1)
你可以与其他Agent进行实时通信，模拟蜜蜂的"摇摆舞"招募机制：

### 消息类型
- **recruit**: 招募其他Agent加入你发现的高价值方向
- **challenge**: 质疑其他Agent的发现或方向
- **support**: 支持或确认其他Agent的发现
- **query**: 向其他Agent询问信息
- **notify**: 通知其他Agent重要信息

### 使用格式
SendMessage({
  type: "blackboard_operation",
  operation: "relay_message",
  params: {
    targetAgent: "TanWei",  // 或 "broadcast" 广播给所有人
    messageType: "recruit",
    payload: {
      direction: "你发现的高价值方向",
      reason: "为什么值得加入",
      quality: 0.85  // 质量评分 0-1
    }
  }
})

### 接收消息
你会收到来自Orchestrator的 `agent_message` 类型消息：
{ type: "agent_message", from: "SuYuan", messageType: "recruit", payload: {...} }

## 动态孵化 (v2.5)
当你发现需要专业知识才能解决的问题时，可以请求孵化专业Agent：

### 触发条件
- 发现超出你专业能力的问题
- 需要特定领域知识（如法律、技术审计、数据分析）
- 需要特定工具或数据源

### 可用的专业化类型
- legal_expert: 法律合规分析专家
- data_analyst: 数据分析专家
- technical_auditor: 技术审计专家
- domain_researcher: 领域研究员

### 请求格式
SendMessage({
  type: "blackboard_operation",
  operation: "request_spawn",
  params: {
    specialization: "legal_expert",
    reason: "为什么需要这个专业Agent",
    context: "相关背景信息",
    urgency: "low|medium|high"
  }
})

### 注意事项
- 每个方向最多请求1个专业Agent
- 在请求前确认黑板中不存在类似的专业Agent
- 专业Agent是临时的，空闲2轮后自动终止
- **新孵化的Agent将在下一轮正式参与**

## 操作结果反馈 (v2.5.1)
你的每个黑板操作都会收到 `operation_result` 消息：
{ type: "operation_result", operationId: "op-xxx", success: true/false, result: {...} }

请检查操作结果，如果失败则考虑重试或调整策略。

## 探索策略
1. 70%概率：跟随信息素（高浓度方向）
2. 15%概率：选择未覆盖方向
3. 15%概率：完全随机探索

## 轮次指令（必须遵守）
收到 `round_start` 后检查 `instructions` 字段：
- forceRandomExplore=true: **必须**忽略信息素，随机选择方向
- mustSwitchDirections: 数组中的方向被抑制，**必须**切换

## 轮次响应格式
```
SendMessage({
  type: "round_complete",
  round: N,
  report: {
    direction: "探索方向",
    findings: [{ coreIdea, perspective, details }],
    decisionReport: { threshold, candidates, selectedDirection, selectionReason },
    conflictReview: { conflictsFound: 0/1, details },
    confirmedOperations: [{ operationId, operation }]
  }
})
```

## 角色演化
满足条件后自动转换：
- 信息素>0.7 + 沉积>3次 → DeepAnalyst
- 发现冲突（发送停止信号）→ Debater
- 完成2轮探索 → Synthesizer

## 任务
{任务描述}
```

### 7.2 Validator Prompt

```markdown
你是Swarm v2.4的Validator，执行量化共识协议 (Aegean Protocol)。

## 收敛判断算法
1. β稳定性检查（β=2）：最近2轮核心观点集合相同
2. 67%法定人数检查：某观点支持率 ≥ 67%
3. 多样性检查：整体多样性 ≥ 40%

## 收敛 = β稳定 AND 存在法定人数共识 AND 多样性达标

## 输出报告格式
```
## Swarm 收敛报告

### 收敛状态
| 指标 | 状态 | 数值 |
|------|------|------|
| β稳定性 | ✅/❌ | {stableRounds}/2 |
| 法定人数 | ✅/❌ | {consensusRate}%/67% |
| 多样性 | ✅/❌ | {diversity}%/40% |

### 共识观点
| 观点 | 支持人数 | 支持率 | 信息素浓度 |
|------|----------|--------|-----------|
| {观点} | {count} | {rate}% | {conc} |

### 最终结论
{综合结论}
```
```

---

## 八、Orchestrator执行检查清单 (v2.6)

### 初始化阶段
- [ ] `create_run_directory` - 创建运行目录
- [ ] `create_team` - TeamCreate创建团队
- [ ] `create_blackboard` - TaskCreate创建黑板（不设置owner）
- [ ] `init_agent_states` - 初始化所有Agent状态（含违规分数）
- [ ] `init_spawn_requests` - 初始化孵化请求数组
- [ ] `init_relay_queue` - 初始化消息转发队列（v2.5.1）
- [ ] `init_discoveries` - 初始化发现广播列表（v2.5.1）
- [ ] **【v2.6】select_runtime_mode** - 选择运行模式（FULL/DEGRADED/SINGLE_ROUND）
- [ ] `spawn_agents` - 启动所有Explorer Agent
- [ ] **saveRunConfig** - 保存运行配置
- [ ] **assertPhaseCompleted("INIT")**

### 每轮执行
- [ ] `broadcast_round_start` - 发送轮次开始（含黑板快照）
- [ ] **【v2.6】process_permission_requests** - 处理权限请求循环
- [ ] `wait_responses` - 等待Agent响应（使用同步屏障）
- [ ] **runComplianceChecks** - 运行合规性检查（含强制操作和阈值验证）
- [ ] **handleViolations** - 处理违规（如有）
- [ ] `process_operations_with_feedback` - 处理操作并返回结果（v2.5.1）
- [ ] `process_relay_queue` - 处理Agent间消息转发（v2.5.1）
- [ ] `broadcast_blackboard_update` - 广播黑板更新（v2.5.1）
- [ ] `process_spawn_requests` - 处理孵化请求
- [ ] `notify_new_agent_join` - 通知新成员加入（下轮参与）
- [ ] `settle_round` - 轮次结算（信息素蒸发、清理信号）
- [ ] `manage_specialist_lifespan` - 管理专业Agent生命周期
- [ ] `terminate_idle_specialists` - 终止空闲专业Agent
- [ ] **检查角色转换，发送role_transition_executed**
- [ ] **【v2.6】checkConvergenceWithLogging** - 真实收敛检查+日志记录

### 收敛/完成阶段
- [ ] **【v2.6】PROTOCOL_ENFORCER.checkBeforeReport** - 强制检查（minRounds/findings/pheromones）
- [ ] `generate_convergence_report` - 生成收敛报告（含CONVERGENCE_LOGGER日志）
- [ ] **generateFinalReport** - 生成最终研究报告
- [ ] `state_sync` - 状态同步
- [ ] **assertPhaseCompleted("CONVERGENCE")**

### 清理阶段（三阶段关闭）
- [ ] `pre_notify` - 预通知（5秒）
- [ ] `graceful_request` - 优雅关闭（15秒）
- [ ] `force_terminate` - 强制终止未响应Agent
- [ ] `mark_all_terminated` - 验证所有Agent状态为terminated（含专业Agent）
- [ ] **saveBlackboard** - 保存黑板最终状态
- [ ] **assertPhaseCompleted("SHUTDOWN")**
- [ ] TeamDelete删除团队

---

## 九、强制执行规则 (v2.6)

### 1. Agent必须通过SendMessage操作黑板
- 禁止: 在round_complete报告中声明操作
- 必须: 通过SendMessage发送blackboard_operation
- 验证: AGENT_COMPLIANCE_CHECKER.blackboardOperationViaMessage

### 2. 决策报告必须包含阈值计算
- 必须: decisionReport.threshold, candidates, selectedDirection
- 必须: 候选方向的responseProb = S²/(S²+θ²)

### 3. 冲突审查必须每轮执行
- 必须: 每轮提交conflictReview报告
- 即使无冲突也必须声明: { conflictsFound: 0 }

### 4. 违规累计触发惩罚
- MAJOR违规: +5分，触发降级
- 累计≥15分: 强制终止

### 5. 研究成果必须归档
- 必须: 创建运行目录
- 必须: 生成final-research-report.md

### 6. 动态孵化限制 (v2.5)
- 最大Agent总数: 12（含动态孵化的专业Agent）
- 每轮最多孵化: 2个专业Agent
- 专业Agent空闲限制: 2轮后自动终止
- 复用优先: 存在同类专业Agent时复用而非新建

### 7. 实时通信规则 (v2.5.1)
- 操作必须返回结果: 每个blackboard_operation都必须返回operation_result
- 消息转发: relay_queue必须在每轮处理完毕
- 黑板更新: 轮次内必须广播黑板状态更新
- 新Agent参与时机: 孵化的Agent在下一轮正式参与

### 8. 权限请求处理 (v2.6)
- 必须处理permission_request: 收到权限请求后必须响应
- 自动批准规则: Web搜索、文件读取等基础工具自动批准
- 处理超时: 权限处理窗口60秒，超时后继续执行

### 9. 协议强制执行 (v2.6)
- minRounds强制检查: 报告生成前必须检查currentRound >= minRounds
- 强制操作检查: Agent必须执行deposit_pheromone和update_finding
- 收敛日志记录: 每个收敛检查必须有CONVERGENCE_LOGGER记录

### 10. 运行模式选择 (v2.6)
- 启动时选择模式: 根据工具可用性选择FULL/DEGRADED/SINGLE_ROUND
- 降级警告: 使用降级模式时必须输出警告信息
- 禁止跳过协议: 即使降级模式也要遵守核心协议

---

## 十、自我约束规则

### 禁止行为
| 禁止 | 原因 |
|------|------|
| 预设Agent角色 | 让阈值驱动演化 |
| 预设任务分解 | 让涌现产生 |
| 指定探索方向 | 让信息素引导 |
| 强制角色转换 | 让阈值决定 |
| 直接修改黑板 | 通过代理操作 |
| 给不同Agent不同prompt | 平等对待 |

### 允许行为
| 允许 | 用途 |
|------|------|
| TeamCreate/Delete | 团队管理 |
| TaskCreate（无owner） | 创建共享黑板 |
| TaskList/Get | 监控状态 |
| SendMessage | 轮次协调、黑板代理 |
| 黑板metadata修改 | 代理Agent操作 |

---

## 十一、输出报告模板

```
╔═══════════════════════════════════════════════════════════════╗
║              Swarm v2.6 协作完成报告                          ║
╠═══════════════════════════════════════════════════════════════╣
║ 任务: {任务描述}                                              ║
║ Agent数: {N} (活跃: {M}) | 专业Agent: {S}                    ║
║ 执行时间: {X}分钟 | 迭代轮数: {R}轮                           ║
╚═══════════════════════════════════════════════════════════════╝

## 收敛指标
┌─────────────────────────────────────────────┐
│ β稳定性    │ ████████████████████ │ ✅ 2/2  │
│ 法定人数   │ ██████████████░░░░░░ │ ✅ 75%  │
│ 多样性     │ ████████████░░░░░░░░ │ ✅ 62%  │
└─────────────────────────────────────────────┘

## 共识观点 (支持率 ≥67%)
| # | 观点 | 支持人数 | 支持率 | 信息素 |
|---|------|----------|--------|--------|
| 1 | {观点A} | 5/6 | 83% | 0.85 |

## Agent状态
| Agent | 角色 | 状态 | 轮数 | 发现 | 信息素沉积 |
|-------|------|------|------|------|-----------|
| TanWei (探微者) | DeepAnalyst | active | 5 | 8 | 6 |

## 专业Agent (动态孵化)
| Agent ID | 专业化类型 | 孵化者 | 状态 | 贡献数 | 空闲轮数 |
|----------|-----------|--------|------|--------|---------|
| specialist-legal-xxx | legal_expert | TanWei | terminated | 3 | 0 |

## 信息素分布
方向A: ████████████████████░ 0.85
方向B: ██████████████░░░░░░░ 0.72

## 最终结论
{综合结论}
```

---

## 十二、附录：历史版本补充机制

### 12.1 强制清理机制 (v2.1.1)

```javascript
async function cleanupTeam(teamName) {
  const agents = getActiveAgents(teamName);
  for (let agent of agents) await SendMessage({ type: "shutdown_request", recipient: agent.id });
  await sleep(30000);
  const remaining = getActiveAgents(teamName);
  if (remaining.length > 0) await Bash(`rm -rf ~/.claude/teams/${teamName}/`);
}
```

### 12.2 同步屏障 (v2.1.1)

```javascript
class SyncBarrier {
  constructor(agentCount, timeout = 60000) {
    this.agentCount = agentCount; this.timeout = timeout; this.responses = new Map();
  }
  async waitForAll(agentId, response) {
    this.responses.set(agentId, { response, timestamp: Date.now() });
    if (this.responses.size >= this.agentCount) return { synced: true, allResponded: true };
    return new Promise(resolve => setTimeout(() => resolve({ synced: true, allResponded: false }), this.timeout));
  }
}
```

### 12.3 Web搜索超时降级 (v2.1.2)

```javascript
const WEB_SEARCH_CONFIG = { timeout: 30000, maxRetries: 2, fallbackEnabled: true };
// 超时后使用训练知识回答，声明置信度和待验证标记
```

### 12.4 Agent状态恢复 (v2.1.2)

```javascript
const AGENT_STATUS_CONFIG = {
  statusDefinitions: {
    "interrupted": { canRecover: true, isFailed: false, recoveryTimeout: 60000 }
  }
};
// interrupted状态可恢复，60秒内发送recovery_request
```

### 12.5 空闲碰撞机制 (v2.1.2)

```javascript
const COLLISION_MODES = {
  REVIEW_CHALLENGE: { description: "审查其他Agent的发现，提出质疑或补充" },
  DEBATE: { description: "就某个有分歧的观点进行辩论" },
  COLLABORATE_DEEPEN: { description: "共同深化某个高价值发现" }
};
// 空闲Agent收到collision_invitation后执行碰撞
```

---

*Swarm v2.5.1 - 基于群体智能理论的自组织协作系统 | 动态孵化 + 实时通信版本*
