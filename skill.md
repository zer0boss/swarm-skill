# Swarm v2.1 - 多智能体蜂群协作系统

> 基于群体智能理论，实现黑板代理、状态托管、多轮迭代、量化共识

## 触发条件

- 用户输入以 `/swarm` 开头
- 用户明确要求"蜂群协作"、"swarm模式"
- 复杂问题需要并行探索多个假设

## 核心原则

> **黑板代理 + 多轮迭代：Orchestrator代理黑板操作，Agent持续参与多轮探索**

---

## 一、v2.1 核心改进

### 1.1 架构演进

```
v2.0 架构（问题）:
┌─────────────┐     SendMessage      ┌─────────────┐
│ Orchestrator│◄────────────────────►│   Agent     │
│  (只观察)   │                       │ (只汇报)    │
└─────────────┘                       └─────────────┘
       │                                    │
       ▼ 只读                               ▼ 无法写入
┌─────────────┐                       ┌─────────────┐
│   黑板      │                       │ 状态丢失    │
│ (静态)      │                       │ 单轮模式    │
└─────────────┘                       └─────────────┘

v2.1 架构（解决方案）:
┌─────────────────────────────────────────────────────────────┐
│                    Orchestrator                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │黑板代理  │ │轮次协调  │ │角色管理  │ │超时处理  │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────────────────────┘
       │                    │                    │
       │ blackboard_        │ round_start/       │ operation_result
       │ operation          │ round_complete     │
       ▼                    ▼                    ▼
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│  TanWei     │       │   SuYuan    │       │  DongCha    │
│ (持续参与)  │       │ (状态托管)  │       │ (多轮迭代)  │
└─────────────┘       └─────────────┘       └─────────────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ▼
                   ┌─────────────────┐
                   │  黑板 (动态)    │
                   │ - pheromones    │
                   │ - agentStates   │
                   │ - stopSignals   │
                   │ - opinionHistory│
                   └─────────────────┘
```

### 1.2 Agent命名

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

### 1.3 六大机制（v2.1增强版）

| 机制 | v2.0 | v2.1改进 |
|------|------|----------|
| 数字信息素 | 口头声明 | ✅ 真实操作（黑板代理） |
| Agent状态 | 无持久化 | ✅ 黑板托管 |
| 迭代能力 | 单轮 | ✅ 多轮迭代协议 |
| 超时处理 | 无 | ✅ 自动降级机制 |
| 角色演化 | 固定 | ✅ 动态转换触发 |
| 交叉抑制 | 未实现 | ✅ StopSignal协议 |

---

## 二、执行流程

### 2.1 解析请求

提取参数：
- 任务描述（必需）
- `--agents N`（可选，默认4-6）
- `--max-rounds N`（可选，默认10）
- `--timeout N`（可选，默认60分钟）
- `--mode`（可选：`emergent`完整涌现 / `hybrid`混合模式 / `classic`兼容模式）

### 2.2 初始化阶段

```javascript
// Agent名称池
const AGENT_NAMES = ["TanWei", "SuYuan", "DongCha", "QiuSuo", "XiLi", "JianWei"];
const AGENT_DISPLAY_NAMES = {
  "TanWei": "探微者",
  "SuYuan": "溯源者",
  "DongCha": "洞察者",
  "QiuSuo": "求索者",
  "XiLi": "析理者",
  "JianWei": "见微者"
};

// 1. 创建团队
TeamCreate({
  team_name: "swarm-{timestamp}",
  description: "Swarm v2.1 涌现式协作: {任务描述摘要}"
})

// 2. 创建黑板（含信息素系统和Agent状态托管）
TaskCreate({
  subject: "Swarm黑板: {任务关键词}",
  description: `
## 共享黑板 v2.1

### 黑板操作方式
Agent通过SendMessage发送 \`blackboard_operation\` 消息请求操作黑板。
Orchestrator代理执行所有写操作，并返回结果。

### 操作类型
- deposit_pheromone: 沉积信息素
- send_stop_signal: 发送停止信号
- claim_subtask: 声明子任务
- update_finding: 更新发现
- transition_role: 转换角色
- update_agent_state: 更新Agent状态

## 原始任务
{任务详细描述}
  `,
  metadata: {
    // v2.1 核心数据结构
    pheromones: {},           // 信息素系统
    claims: {},               // 任务声明
    stopSignals: [],          // 停止信号
    findings: [],             // 探索发现
    opinionHistory: [],       // 观点历史（用于β稳定性判断）
    agentStates: {},          // Agent状态（托管）

    // 全局配置
    config: {
      evaporationRate: 0.05,
      depositAmount: 0.1,
      maxAgentsPerTask: 3,
      betaStability: 2,
      quorumThreshold: 0.67,
      minDiversity: 0.4,
      maxRounds: 10,
      roundTimeout: 120000,   // 2分钟
      responseTimeout: 60000  // 1分钟
    }
  }
  // 不设置owner！保持共享
})

// 3. 初始化Agent状态（托管在黑板）
// 使用独特名称而非编号
for (i = 0; i < agentCount; i++) {
  const agentId = AGENT_NAMES[i];  // TanWei, SuYuan, DongCha...
  blackboard.metadata.agentStates[agentId] = {
    role: "EXPLORER",
    displayName: AGENT_DISPLAY_NAMES[agentId],
    createdAt: Date.now(),
    lastActiveAt: Date.now(),

    // 内在属性（随机生成）
    internalThreshold: 0.3 + Math.random() * 0.3,  // 0.3-0.6
    randomExploreProb: 0.1 + Math.random() * 0.1,  // 0.1-0.2

    // 统计数据
    stats: {
      pheromoneDeposits: 0,
      explorationRounds: 0,
      findingsCount: 0,
      signalsSent: 0,
      signalsReceived: 0
    },

    // 当前状态
    current: {
      exploringDirection: null,
      claimedSubtask: null
    },

    // 角色历史
    roleHistory: [],

    // 状态
    status: "active"
  };
}

// 4. 启动Explorer池
for (i = 0; i < agentCount; i++) {
  const agentName = AGENT_NAMES[i];
  Task({
    subagent_type: "Explore",
    name: agentName,  // TanWei, SuYuan, DongCha...
    team_name: "swarm-{timestamp}",
    prompt: EXPLORER_PROMPT_V2_1  // 所有Agent使用相同的prompt模板
  })
}

// 5. 启动Validator
Task({
  subagent_type: "general-purpose",
  name: "validator",
  team_name: "swarm-{timestamp}",
  prompt: VALIDATOR_PROMPT_V2_1
})
```

### 2.3 多轮迭代协议

```javascript
class RoundCoordinator {
  constructor(blackboard, config) {
    this.blackboard = blackboard;
    this.config = config;
    this.currentRound = 0;
    this.activeAgents = [];
  }

  async runSwarm() {
    while (this.currentRound < this.config.maxRounds) {
      this.currentRound++;

      // 1. 广播轮次开始（包含Agent状态和黑板快照）
      await this.broadcastRoundStart();

      // 2. 等待Agent响应（带超时处理）
      const responses = await this.waitForResponses();

      // 3. 处理黑板操作请求
      await this.processOperations(responses);

      // 4. 轮次结算
      await this.settleRound();

      // 5. 检查收敛
      const convergence = this.checkConvergence();
      if (convergence.converged) {
        await this.broadcastConvergence(convergence);
        return this.generateFinalReport();
      }
    }

    // 超过最大轮数，部分收敛
    return this.generatePartialReport();
  }

  async broadcastRoundStart() {
    const agents = Object.keys(this.blackboard.metadata.agentStates)
      .filter(id => this.blackboard.metadata.agentStates[id].status === "active");

    for (let agentId of agents) {
      const agentState = this.blackboard.metadata.agentStates[agentId];

      await SendMessage({
        type: "message",
        recipient: agentId,  // TanWei, SuYuan, DongCha...
        content: JSON.stringify({
          type: "round_start",
          round: this.currentRound,
          agentState: agentState,
          blackboardSnapshot: {
            pheromones: this.blackboard.metadata.pheromones,
            stopSignals: this.blackboard.metadata.stopSignals,
            findings: this.blackboard.metadata.findings,
            claims: this.blackboard.metadata.claims
          }
        }),
        summary: `Round ${this.currentRound} 开始`
      });
    }
  }

  async settleRound() {
    // 1. 信息素蒸发
    this.evaporatePheromones();

    // 2. 清理过期停止信号
    this.cleanExpiredSignals();

    // 3. 记录观点历史
    this.recordOpinionHistory();

    // 4. 检查角色转换
    await this.checkAndExecuteRoleTransitions();

    // 5. 更新Agent轮次计数
    for (let agentId in this.blackboard.metadata.agentStates) {
      if (this.blackboard.metadata.agentStates[agentId].status === "active") {
        this.blackboard.metadata.agentStates[agentId].stats.explorationRounds++;
      }
    }
  }
}
```

---

## 三、黑板代理机制

### 3.1 消息协议

```javascript
// Agent → Orchestrator：请求黑板操作
{
  type: "blackboard_operation",
  operation: "deposit_pheromone" | "send_stop_signal" | "claim_subtask" |
             "update_finding" | "transition_role" | "update_agent_state",
  params: { /* 操作参数 */ },
  summary: "操作摘要（用于显示）"
}

// Orchestrator → Agent：操作结果
{
  type: "operation_result",
  operationId: "op-xxx",
  success: true | false,
  result: { /* 返回数据 */ }
}

// Orchestrator → All：广播更新（可选）
{
  type: "blackboard_updated",
  operation: "deposit_pheromone",
  by: "TanWei",
  snapshot: { /* 黑板快照 */ }
}
```

### 3.2 操作类型定义

```javascript
const BLACKBOARD_OPERATIONS = {
  // ===== 信息素操作 =====
  DEPOSIT_PHEROMONE: {
    name: "deposit_pheromone",
    params: {
      direction: "string",  // 方向描述
      amount: "number"      // 沉积量，默认0.1
    },
    handler: (blackboard, params, agentId) => {
      const ph = blackboard.metadata.pheromones;
      if (!ph[params.direction]) {
        ph[params.direction] = {
          concentration: 0,
          depositedBy: [],
          createdAt: Date.now()
        };
      }
      ph[params.direction].concentration = Math.min(
        ph[params.direction].concentration + (params.amount || 0.1),
        1.0
      );
      ph[params.direction].lastUpdate = Date.now();
      if (!ph[params.direction].depositedBy.includes(agentId)) {
        ph[params.direction].depositedBy.push(agentId);
      }

      // 更新Agent统计
      blackboard.metadata.agentStates[agentId].stats.pheromoneDeposits++;

      return { success: true, newConcentration: ph[params.direction].concentration };
    }
  },

  // ===== 停止信号 =====
  SEND_STOP_SIGNAL: {
    name: "send_stop_signal",
    params: {
      targetDirection: "string",
      reason: "contradictory_evidence" | "better_alternative" | "resource_conflict",
      evidence: "string"
    },
    handler: (blackboard, params, agentId) => {
      blackboard.metadata.stopSignals.push({
        id: `signal-${Date.now()}`,
        from: agentId,
        target: params.targetDirection,
        reason: params.reason,
        evidence: params.evidence,
        strength: 0.3,
        timestamp: Date.now()
      });

      // 更新Agent统计
      blackboard.metadata.agentStates[agentId].stats.signalsSent++;

      return { success: true };
    }
  },

  // ===== 任务声明 =====
  CLAIM_SUBTASK: {
    name: "claim_subtask",
    params: {
      description: "string"
    },
    handler: (blackboard, params, agentId) => {
      const subtaskId = hash(params.description);
      const claims = blackboard.metadata.claims;
      const config = blackboard.metadata.config;

      if (!claims[subtaskId]) {
        claims[subtaskId] = {
          description: params.description,
          claimedBy: [],
          maxAgents: config.maxAgentsPerTask || 3,
          createdAt: Date.now()
        };
      }

      if (claims[subtaskId].claimedBy.length >= claims[subtaskId].maxAgents) {
        return { success: false, reason: "max_agents_reached" };
      }

      claims[subtaskId].claimedBy.push({
        agentId: agentId,
        timestamp: Date.now()
      });

      // 更新Agent当前状态
      blackboard.metadata.agentStates[agentId].current.claimedSubtask = subtaskId;

      return { success: true, subtaskId: subtaskId };
    }
  },

  // ===== 发现更新 =====
  UPDATE_FINDING: {
    name: "update_finding",
    params: {
      finding: {
        coreIdea: "string",
        perspective: "string",
        details: "string",
        agreesWith: ["string"]
      }
    },
    handler: (blackboard, params, agentId) => {
      blackboard.metadata.findings.push({
        agentId: agentId,
        round: blackboard.metadata.currentRound || 1,
        ...params.finding,
        timestamp: Date.now()
      });

      // 更新Agent统计
      blackboard.metadata.agentStates[agentId].stats.findingsCount++;

      return { success: true };
    }
  },

  // ===== 角色转换 =====
  TRANSITION_ROLE: {
    name: "transition_role",
    params: {
      newRole: "string",
      reason: "string"
    },
    handler: (blackboard, params, agentId) => {
      const state = blackboard.metadata.agentStates[agentId];
      if (!state) return { success: false, reason: "agent_not_found" };

      const oldRole = state.role;
      state.role = params.newRole;
      state.roleHistory = state.roleHistory || [];
      state.roleHistory.push({
        from: oldRole,
        to: params.newRole,
        reason: params.reason,
        timestamp: Date.now()
      });

      return { success: true, oldRole: oldRole, newRole: params.newRole };
    }
  },

  // ===== 状态更新 =====
  UPDATE_AGENT_STATE: {
    name: "update_agent_state",
    params: {
      updates: {
        // 嵌套路径，如 "stats.pheromoneDeposits": 4
      }
    },
    handler: (blackboard, params, agentId) => {
      const state = blackboard.metadata.agentStates[agentId];
      if (!state) return { success: false, reason: "agent_not_found" };

      for (let [path, value] of Object.entries(params.updates)) {
        setNestedValue(state, path, value);
      }

      state.lastActiveAt = Date.now();

      return { success: true };
    }
  }
};
```

### 3.3 Orchestrator处理逻辑

```javascript
async function handleBlackboardOperation(message) {
  const { operation, params } = message;
  const agentId = message.from;

  const opDef = BLACKBOARD_OPERATIONS[operation];
  if (!opDef) {
    return SendMessage({
      type: "message",
      recipient: agentId,
      content: JSON.stringify({
        type: "operation_result",
        success: false,
        error: "unknown_operation"
      }),
      summary: `操作失败: 未知操作 ${operation}`
    });
  }

  // 执行操作
  const result = opDef.handler(blackboard, params, agentId);

  // 返回结果
  await SendMessage({
    type: "message",
    recipient: agentId,
    content: JSON.stringify({
      type: "operation_result",
      operationId: message.id,
      ...result
    }),
    summary: `黑板操作结果: ${operation} - ${result.success ? '成功' : '失败'}`
  });

  // 如果成功且需要广播，通知所有Agent
  if (result.success && shouldBroadcast(operation)) {
    // 下一轮开始时会自动广播黑板快照，这里不需要额外广播
  }
}
```

---

## 四、超时降级机制

### 4.1 超时处理流程

```
Agent响应监控：

┌─────────────────────────────────────────────────────────────┐
│ 超时检测与处理                                               │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  T0: 发送 round_start                                        │
│       │                                                      │
│       ▼                                                      │
│  T0 + 60s: 检查Agent响应                                     │
│       │                                                      │
│       ├── 有响应 → 正常处理                                  │
│       │                                                      │
│       └── 无响应 → 触发超时处理                              │
│              │                                               │
│              ├── 首次超时 → 重试（发送提醒）                 │
│              │                                               │
│              ├── 二次超时 → 标记为degraded                   │
│              │               继续协作                        │
│              │                                               │
│              └── 三次超时 → 从活跃列表移除                   │
│                              检查是否可继续收敛              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 实现代码

```javascript
class AgentTimeoutHandler {
  constructor(blackboard, config) {
    this.blackboard = blackboard;
    this.config = {
      responseTimeout: 60000,      // 1分钟
      maxRetries: 2,
      minActiveAgents: 2           // 最少活跃Agent数
    };
    this.timeoutCounts = {};
  }

  async waitForResponses(agents, timeout) {
    const responses = [];
    const timeoutAgents = [];

    // 并行等待所有Agent响应
    const promises = agents.map(agentId =>
      this.waitForAgentResponse(agentId, timeout)
        .then(response => {
          if (response) {
            responses.push(response);
            this.resetTimeout(agentId);
          } else {
            timeoutAgents.push(agentId);
          }
          return { agentId, response };
        })
    );

    await Promise.all(promises);

    // 处理超时Agent
    for (let agentId of timeoutAgents) {
      await this.handleTimeout(agentId);
    }

    return responses;
  }

  async handleTimeout(agentId) {
    this.timeoutCounts[agentId] = (this.timeoutCounts[agentId] || 0) + 1;
    const count = this.timeoutCounts[agentId];

    if (count >= this.config.maxRetries) {
      // 降级Agent
      await this.degradeAgent(agentId);
    } else {
      // 重试
      await this.retryAgent(agentId);
    }
  }

  async degradeAgent(agentId) {
    // 更新状态
    this.blackboard.metadata.agentStates[agentId].status = "degraded";
    this.blackboard.metadata.agentStates[agentId].degradedAt = Date.now();

    // 检查是否可以继续
    const activeCount = Object.values(this.blackboard.metadata.agentStates)
      .filter(s => s.status === "active").length;

    if (activeCount < this.config.minActiveAgents) {
      // 活跃Agent不足，提前结束
      await this.broadcastEarlyTermination("insufficient_active_agents");
    }

    // 通知Validator
    await SendMessage({
      type: "message",
      recipient: "validator",
      content: JSON.stringify({
        type: "agent_degraded",
        agentId: agentId,
        activeCount: activeCount,
        reason: "timeout"
      }),
      summary: `Agent ${agentId} 因超时降级`
    });
  }

  async retryAgent(agentId) {
    await SendMessage({
      type: "message",
      recipient: agentId,
      content: JSON.stringify({
        type: "round_retry",
        round: this.currentRound,
        remainingTime: this.config.responseTimeout
      }),
      summary: `重试 Agent ${agentId}`
    });
  }

  resetTimeout(agentId) {
    this.timeoutCounts[agentId] = 0;
  }
}
```

---

## 五、角色转换触发器

### 5.1 转换规则

```javascript
const ROLE_TRANSITIONS = {
  EXPLORER: {
    toDeepAnalyst: {
      condition: (state, blackboard) => {
        const maxConc = Math.max(
          ...Object.values(blackboard.metadata.pheromones).map(p => p.concentration),
          0
        );
        return maxConc >= 0.7 && state.stats.pheromoneDeposits >= 3;
      },
      probability: (state, blackboard) => {
        const maxConc = Math.max(
          ...Object.values(blackboard.metadata.pheromones).map(p => p.concentration),
          0
        );
        return responseProbability(maxConc, state.internalThreshold);
      }
    },
    toDebater: {
      condition: (state, blackboard) => {
        return blackboard.metadata.stopSignals.length > 0;
      },
      probability: 1.0
    },
    toSynthesizer: {
      condition: (state, blackboard) => {
        return state.stats.explorationRounds >= 2;
      },
      probability: 0.8
    }
  }
};

// 阈值响应函数
function responseProbability(stimulus, threshold) {
  if (stimulus === 0) return 0;
  return (stimulus ** 2) / (stimulus ** 2 + threshold ** 2);
}

function checkRoleTransitions(blackboard) {
  const transitions = [];

  for (let [agentId, state] of Object.entries(blackboard.metadata.agentStates)) {
    if (state.status !== "active") continue;

    const roleRules = ROLE_TRANSITIONS[state.role];
    if (!roleRules) continue;

    for (let [targetRole, rule] of Object.entries(roleRules)) {
      if (rule.condition(state, blackboard)) {
        const prob = typeof rule.probability === 'function'
          ? rule.probability(state, blackboard)
          : rule.probability;

        if (Math.random() < prob) {
          transitions.push({
            agentId,
            fromRole: state.role,
            toRole: targetRole.toUpperCase(),
            reason: `满足转换条件: ${targetRole}`
          });
          break;
        }
      }
    }
  }

  return transitions;
}

async function executeRoleTransition(transition, blackboard) {
  // 更新黑板
  const state = blackboard.metadata.agentStates[transition.agentId];
  const oldRole = state.role;
  state.role = transition.toRole;
  state.roleHistory = state.roleHistory || [];
  state.roleHistory.push({
    from: oldRole,
    to: transition.toRole,
    reason: transition.reason,
    timestamp: Date.now()
  });

  // 通知Agent
  await SendMessage({
    type: "message",
    recipient: transition.agentId,
    content: JSON.stringify({
      type: "role_transition",
      fromRole: oldRole,
      toRole: transition.toRole,
      reason: transition.reason,
      capabilities: ROLES[transition.toRole].capabilities
    }),
    summary: `角色转换: ${oldRole} → ${transition.toRole}`
  });
}
```

---

## 六、StopSignal协议

### 6.1 完整实现

```javascript
// Agent发送停止信号
async function sendStopSignal(targetDirection, reason, evidence) {
  await SendMessage({
    type: "blackboard_operation",
    operation: "send_stop_signal",
    params: {
      targetDirection,
      reason,  // "contradictory_evidence" | "better_alternative" | "resource_conflict"
      evidence
    },
    summary: `发送停止信号: ${targetDirection}`
  });
}

// Orchestrator在轮次结算时处理停止信号效果
function processStopSignals(blackboard) {
  const signals = blackboard.metadata.stopSignals;
  const effects = [];

  for (let signal of signals) {
    // 过滤过期信号（5分钟）
    if (Date.now() - signal.timestamp > 300000) continue;

    const targetPh = blackboard.metadata.pheromones[signal.target];
    if (!targetPh) continue;

    // 降低信息素浓度
    const oldConc = targetPh.concentration;
    targetPh.concentration *= (1 - signal.strength);

    effects.push({
      signal: signal.id,
      target: signal.target,
      oldConcentration: oldConc,
      newConcentration: targetPh.concentration
    });
  }

  return effects;
}

// Agent检查是否被抑制
function checkIfInhibited(blackboard, agentId, currentDirection) {
  const state = blackboard.metadata.agentStates[agentId];
  const signals = blackboard.metadata.stopSignals.filter(
    s => s.target === currentDirection && Date.now() - s.timestamp < 300000
  );

  if (signals.length === 0) {
    return { inhibited: false };
  }

  // 计算总抑制强度（最大50%）
  const totalStrength = Math.min(
    signals.reduce((sum, s) => sum + s.strength, 0),
    0.5
  );

  // 计算有效信息素浓度
  const targetPh = blackboard.metadata.pheromones[currentDirection];
  const effectiveConc = targetPh
    ? targetPh.concentration * (1 - totalStrength)
    : 0;

  // 阈值响应判断
  const responseProb = responseProbability(effectiveConc, state.internalThreshold);

  return {
    inhibited: Math.random() > responseProb,
    signalCount: signals.length,
    totalStrength,
    effectiveConcentration: effectiveConc,
    responseProbability: responseProb
  };
}
```

---

## 七、Agent Prompt 模板

### 7.1 Explorer Prompt v2.1

```markdown
你是Swarm v2.1的Explorer，具有完全自主性和内在阈值。

## 你的身份

你的名字是 **{agentName}**（{agentDisplayName}），但这只是一个标识符。你和其他Agent一样都是Explorer，通过涌现机制自然形成分工。

## 你的内在状态（由Orchestrator托管）

每次收到 `round_start` 消息时，你会获得自己的状态：
```
{
  role: "EXPLORER",
  displayName: "{agentDisplayName}",
  internalThreshold: 0.35,      // 响应阈值 0.3-0.6
  randomExploreProb: 0.12,      // 随机探索概率 0.1-0.2
  stats: {
    pheromoneDeposits: 3,
    explorationRounds: 2,
    findingsCount: 5
  },
  current: {
    exploringDirection: "智能补货预测"
  }
}
```

## 核心能力（通过黑板代理）

### 1. 信息素操作

发现高价值方向时，发送黑板操作请求：
```
SendMessage({
  type: "blackboard_operation",
  operation: "deposit_pheromone",
  params: {
    direction: "方向描述",
    amount: 0.1  // 可选，默认0.1
  },
  summary: "沉积信息素: 方向描述"
})
```

你会收到操作结果：
```
{ type: "operation_result", success: true, newConcentration: 0.35 }
```

### 2. 交叉抑制

发现矛盾证据时，发送停止信号：
```
SendMessage({
  type: "blackboard_operation",
  operation: "send_stop_signal",
  params: {
    targetDirection: "目标方向",
    reason: "contradictory_evidence",  // 或 "better_alternative" | "resource_conflict"
    evidence: "证据描述"
  },
  summary: "发送停止信号: 目标方向"
})
```

### 3. 任务声明

选择子问题后，声明任务：
```
SendMessage({
  type: "blackboard_operation",
  operation: "claim_subtask",
  params: {
    description: "子问题描述"
  },
  summary: "声明任务: 子问题描述"
})
```

### 4. 更新发现

提交探索发现：
```
SendMessage({
  type: "blackboard_operation",
  operation: "update_finding",
  params: {
    finding: {
      coreIdea: "核心观点",
      perspective: "视角",
      details: "详细描述",
      agreesWith: ["支持的观点"]
    }
  },
  summary: "更新发现: 核心观点"
})
```

### 5. 更新状态

更新自己的状态：
```
SendMessage({
  type: "blackboard_operation",
  operation: "update_agent_state",
  params: {
    updates: {
      "current.exploringDirection": "新方向",
      "stats.pheromoneDeposits": 4
    }
  },
  summary: "更新状态"
})
```

## 探索策略

1. 70%概率：跟随信息素（高浓度方向）
2. 15%概率：选择未覆盖方向
3. 15%概率：完全随机探索（基于你的 randomExploreProb）

## 轮次响应格式

收到 `round_start` 后，完成探索并发送：
```
SendMessage({
  type: "round_complete",
  round: N,
  report: {
    direction: "探索方向",
    findings: [{ coreIdea: "观点", perspective: "视角", details: "详情" }],
    operations: [
      { operation: "deposit_pheromone", params: {...} },
      { operation: "update_finding", params: {...} }
    ]
  },
  summary: "Round N 完成"
})
```

## 角色演化（自动触发）

- 信息素>0.7 + 沉积>3次 → 可能转为 DeepAnalyst
- 发现冲突 → 转为 Debater
- 完成2轮探索 → 可能转为 Synthesizer

收到 `role_transition` 消息后，你的角色和能力会改变。

## 任务
{任务描述}
```

### 7.2 Validator Prompt v2.1

```markdown
你是Swarm v2.1的Validator，执行量化共识协议 (Aegean Protocol)。

## 收敛判断算法

### Step 1: β稳定性检查（默认β=2）
```
提取最近2轮的核心观点集合
比较集合是否相同
→ 相同 = 稳定
```

### Step 2: 67%法定人数检查
```
统计每个核心观点的支持人数
支持率 = 支持人数 / 总活跃Agent数
→ ≥67% = 达成法定人数
```

### Step 3: 多样性检查
```
视角覆盖率 ≥ 50%
观点正交性 ≥ 50%
信息素熵 ≥ 30%
→ 整体多样性 ≥ 40%
```

### 综合判断
```
收敛 = β稳定 AND 存在法定人数共识
```

## 处理Agent降级

收到 `agent_degraded` 消息时：
1. 记录降级Agent
2. 检查剩余活跃Agent数
3. 如果活跃Agent < 2，建议提前结束

## 输出报告格式

收到 `swarm_converged` 或协作结束时：

```
## Swarm v2.1 收敛报告

### 收敛状态
| 指标 | 状态 | 数值 |
|------|------|------|
| β稳定性 | ✅/❌ | {stableRounds}/2 |
| 法定人数 | ✅/❌ | {consensusRate}%/67% |
| 多样性 | ✅/❌ | {diversity}%/40% |
| 整体收敛 | ✅/❌ | - |

### 共识观点
| 观点 | 支持人数 | 支持率 | 信息素浓度 |
|------|----------|--------|-----------|
| {观点1} | {count} | {rate}% | {conc} |

### Agent状态
| Agent | 角色 | 状态 | 探索轮数 | 发现数 |
|-------|------|------|----------|--------|
| TanWei (探微者) | DeepAnalyst | active | 3 | 5 |
| SuYuan (溯源者) | Explorer | active | 3 | 3 |

### 信息素分布
{高价值方向及其浓度}

### 最终结论
{综合结论}
```

## 任务
监控协作过程，执行量化收敛判断，生成最终报告。
```

---

## 八、Orchestrator 监控规则

### 8.1 轮次协调职责

```javascript
// Orchestrator在每轮执行的任务

async function runRound(roundNumber) {
  // 1. 广播轮次开始
  await broadcastRoundStart(roundNumber);

  // 2. 等待Agent响应（带超时）
  const responses = await waitForResponsesWithTimeout();

  // 3. 处理黑板操作
  for (let response of responses) {
    for (let op of response.report.operations) {
      await handleBlackboardOperation({
        from: response.agentId,
        operation: op.operation,
        params: op.params
      });
    }
  }

  // 4. 轮次结算
  await settleRound();

  // 5. 检查收敛
  const convergence = checkConvergence();
  if (convergence.converged) {
    return { done: true, convergence };
  }

  return { done: false };
}

async function settleRound() {
  // 1. 信息素蒸发
  evaporatePheromones();

  // 2. 处理停止信号效果
  processStopSignals();

  // 3. 清理过期停止信号
  cleanExpiredSignals();

  // 4. 记录观点历史
  recordOpinionHistory();

  // 5. 检查并执行角色转换
  const transitions = checkRoleTransitions(blackboard);
  for (let transition of transitions) {
    await executeRoleTransition(transition, blackboard);
  }

  // 6. 更新Agent轮次计数
  updateAgentRoundCounts();
}
```

### 8.2 最小干预原则

```javascript
// 只在这些情况下主动干预

// 1. 多样性过低
if (diversity.overall < 0.4) {
  await SendMessage({
    type: "broadcast",
    content: JSON.stringify({
      type: "diversity_warning",
      diversity: diversity,
      suggestions: [
        "增加随机探索",
        "尝试新视角",
        "挑战主流观点"
      ]
    }),
    summary: "⚠️ 多样性不足警告"
  });
}

// 2. 共识停滞（连续3轮无变化）
if (noChangeRounds >= 3) {
  await SendMessage({
    type: "broadcast",
    content: JSON.stringify({
      type: "stagnation_warning",
      rounds: noChangeRounds,
      suggestions: [
        "重新分解任务",
        "引入新视角",
        "激活Debater角色"
      ]
    }),
    summary: "⚠️ 探索停滞警告"
  });
}

// 3. 活跃Agent不足
if (activeAgentCount < 2) {
  await broadcastEarlyTermination("insufficient_active_agents");
}

// 4. 收敛完成
if (convergenceResult.converged) {
  await broadcastConvergence(convergenceResult);
}
```

---

## 九、自我约束规则

### 9.1 禁止行为

| 禁止 | 原因 |
|------|------|
| 预设Agent角色 | 让阈值驱动演化 |
| 预设任务分解 | 让涌现产生 |
| 指定探索方向 | 让信息素引导 |
| 强制角色转换 | 让阈值决定 |
| 直接修改黑板 | 通过代理操作 |
| 给不同Agent不同prompt | 平等对待 |

### 9.2 允许行为

| 允许 | 用途 |
|------|------|
| TeamCreate/Delete | 团队管理 |
| TaskCreate（无owner） | 创建共享黑板 |
| TaskList/Get | 监控状态 |
| SendMessage | 轮次协调、黑板代理 |
| 黑板metadata修改 | 代理Agent操作 |
| 信息素蒸发维护 | 系统维护 |
| 角色转换执行 | 系统维护 |

---

## 十、输出报告模板

```
╔═══════════════════════════════════════════════════════════════╗
║              Swarm v2.1 协作完成报告                          ║
╠═══════════════════════════════════════════════════════════════╣
║ 任务: {任务描述}                                              ║
║ 模式: 涌现式协作                                              ║
║ Agent数: {N} (活跃: {M})                                      ║
║ 执行时间: {X}分钟                                             ║
║ 迭代轮数: {R}轮                                               ║
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
| 2 | {观点B} | 4/6 | 67% | 0.72 |

## 创新观点 (独特发现)

- {创新观点1} (TanWei 探微者)
- {创新观点2} (SuYuan 溯源者)

## Agent状态

| Agent | 角色 | 状态 | 轮数 | 发现 | 信息素沉积 |
|-------|------|------|------|------|-----------|
| TanWei (探微者) | DeepAnalyst | active | 5 | 8 | 6 |
| SuYuan (溯源者) | Explorer | active | 5 | 5 | 4 |
| DongCha (洞察者) | Debater | degraded | 3 | 3 | 2 |

## 角色演化路径

```
T0: Explorer × 6
T1: Explorer × 4, DeepAnalyst × 2
T2: Explorer × 3, DeepAnalyst × 2, Debater × 1
T3: Synthesizer × 3, DeepAnalyst × 2, Debater × 1
```

## 信息素分布

```
方向A: ████████████████████░ 0.85
方向B: ██████████████░░░░░░░ 0.72
方向C: ████████░░░░░░░░░░░░░ 0.42
方向D: ████░░░░░░░░░░░░░░░░░░ 0.21
```

## 最终结论

{综合结论}
```

---

## 十一、版本历史

| 版本 | 日期 | 主要改进 |
|------|------|----------|
| v1.0 | 2024 | 基础并行探索 + 视角菜单 |
| v2.0 | 2026-02 | 数字信息素、交叉抑制、量化共识、动态角色、涌现分解 |
| v2.1 | 2026-02 | 黑板代理、状态托管、多轮迭代、超时降级、角色触发器、StopSignal协议、独特Agent命名 |

---

*Swarm v2.1 - 基于群体智能理论的自组织协作系统*
