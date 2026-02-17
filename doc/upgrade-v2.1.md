# Swarm v2.1 升级方案

> 基于v2.0协作复盘问题，提出架构级改进

**状态: ✅ 已完成 (2026-02-17)**

---

## 升级完成摘要

### 已实现的功能

| 功能 | 状态 | 说明 |
|------|:----:|------|
| 黑板代理机制 | ✅ | Agent通过SendMessage请求黑板操作，Orchestrator代理执行 |
| Agent状态托管 | ✅ | 状态存储在黑板metadata.agentStates |
| 多轮迭代协议 | ✅ | RoundCoordinator管理轮次生命周期 |
| 超时降级机制 | ✅ | AgentTimeoutHandler处理超时Agent |
| 角色转换触发器 | ✅ | 基于阈值和条件的自动角色转换 |
| StopSignal协议 | ✅ | 完整的交叉抑制实现 |

### 主要改动

1. **架构图更新**: 展示v2.0问题和v2.1解决方案的对比
2. **消息协议新增**: `blackboard_operation`, `operation_result`, `round_start`, `round_complete`
3. **Agent Prompt重写**: 明确黑板代理API使用方式
4. **Orchestrator职责扩展**: 轮次协调、黑板代理、超时处理

---

## 一、问题诊断

### 1.1 协作复盘认定的问题

| # | 问题 | 严重度 | 根因 |
|---|------|:------:|------|
| 1 | Agent无法写入黑板metadata | 🔴高 | 架构限制 |
| 2 | Agent状态无法跨轮次保持 | 🔴高 | 无持久化机制 |
| 3 | 单轮对话模式，无法迭代 | 🔴高 | 缺少轮次协议 |
| 4 | 信息素只是口头声明 | 🟡中 | 无法写入黑板 |
| 5 | Agent超时无处理 | 🟡中 | 无超时机制 |
| 6 | 角色演化未执行 | 🟡中 | 无触发机制 |
| 7 | 交叉抑制未触发 | 🟢低 | 无协议定义 |

### 1.2 根本原因

```
当前架构限制：

┌─────────────────────────────────────────────────────────────┐
│ Agent能力边界                                                │
├─────────────────────────────────────────────────────────────┤
│ ✅ 可以：SendMessage发送消息                                 │
│ ✅ 可以：读取黑板Task内容                                    │
│ ❌ 不能：直接修改黑板metadata                                │
│ ❌ 不能：持久化自身状态                                      │
│ ❌ 不能：持续参与多轮迭代                                    │
│ ❌ 不能：监听黑板变化                                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ Orchestrator能力边界                                         │
├─────────────────────────────────────────────────────────────┤
│ ✅ 可以：创建/删除团队和任务                                 │
│ ✅ 可以：启动Agent                                           │
│ ✅ 可以：SendMessage通信                                     │
│ ✅ 可以：读取收件箱                                          │
│ ❌ 不能：直接控制Agent行为                                   │
│ ❌ 不能：强制关闭Agent（只能请求）                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、升级方案总览

### 2.1 方案矩阵

| 方案 | 解决问题 | 优先级 | 复杂度 |
|------|----------|:------:|:------:|
| 黑板代理机制 | #1, #4 | P0 | 中 |
| Agent状态托管 | #2 | P0 | 低 |
| 多轮迭代协议 | #3 | P0 | 高 |
| 超时降级机制 | #5 | P1 | 中 |
| 角色转换触发器 | #6 | P2 | 中 |
| StopSignal协议 | #7 | P2 | 低 |

### 2.2 架构演进

```
v2.0 架构:
┌─────────────┐     SendMessage      ┌─────────────┐
│ Orchestrator│◄────────────────────►│   Agent     │
│  (只观察)   │                       │ (只汇报)    │
└─────────────┘                       └─────────────┘
       │
       ▼ 只读
┌─────────────┐
│   黑板      │
│ (静态)      │
└─────────────┘

v2.1 架构:
┌─────────────────────────────────────────────────────────────┐
│                    Orchestrator                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │黑板代理  │ │轮次协调  │ │角色管理  │ │超时处理  │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────────────────────┘
       │                    │                    │
       │ blackboard_        │ round_start/       │ role_transition
       │ operation          │ round_complete     │
       ▼                    ▼                    ▼
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Agent-1   │       │   Agent-2   │       │   Agent-N   │
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

---

## 三、P0方案详解

### 3.1 方案一：黑板代理机制

#### 设计思路

Agent无法直接修改黑板，通过Orchestrator代理执行所有写操作。

#### 消息协议

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

// Orchestrator → All：广播更新
{
  type: "blackboard_updated",
  operation: "deposit_pheromone",
  by: "explorer-1",
  snapshot: { /* 黑板快照 */ }
}
```

#### 操作类型定义

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
        ph[params.direction] = { concentration: 0, depositedBy: [], createdAt: Date.now() };
      }
      ph[params.direction].concentration = Math.min(
        ph[params.direction].concentration + (params.amount || 0.1),
        1.0
      );
      ph[params.direction].lastUpdate = Date.now();
      if (!ph[params.direction].depositedBy.includes(agentId)) {
        ph[params.direction].depositedBy.push(agentId);
      }
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

      if (!claims[subtaskId]) {
        claims[subtaskId] = {
          description: params.description,
          claimedBy: [],
          maxAgents: blackboard.metadata.config.maxAgentsPerTask || 3
        };
      }

      if (claims[subtaskId].claimedBy.length >= claims[subtaskId].maxAgents) {
        return { success: false, reason: "max_agents_reached" };
      }

      claims[subtaskId].claimedBy.push({
        agentId: agentId,
        timestamp: Date.now()
      });

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
        agreesWith: ["string"]  // 支持的观点
      }
    },
    handler: (blackboard, params, agentId) => {
      blackboard.metadata.findings.push({
        agentId: agentId,
        ...params.finding,
        timestamp: Date.now()
      });
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

      return { success: true };
    }
  }
};
```

#### Orchestrator处理逻辑

```javascript
async function handleBlackboardOperation(message) {
  const { operation, params } = message;
  const agentId = message.from;

  const opDef = BLACKBOARD_OPERATIONS[operation];
  if (!opDef) {
    return sendResult(message.from, {
      type: "operation_result",
      success: false,
      error: "unknown_operation"
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
    summary: `黑板操作结果: ${operation}`
  });

  // 如果成功，广播更新
  if (result.success && shouldBroadcast(operation)) {
    await SendMessage({
      type: "broadcast",
      content: JSON.stringify({
        type: "blackboard_updated",
        operation: operation,
        by: agentId,
        snapshot: getBlackboardSnapshot()
      }),
      summary: `黑板更新: ${operation} by ${agentId}`
    });
  }
}
```

---

### 3.2 方案二：Agent状态托管

#### 设计思路

Agent状态存储在黑板，每次轮次开始时Agent读取并恢复状态。

#### 状态结构

```javascript
// 黑板 metadata.agentStates 结构
{
  "explorer-1": {
    // 基础信息
    role: "EXPLORER",
    createdAt: 1234567890,
    lastActiveAt: 1234567900,

    // 内在属性（初始化时随机生成，后续不变）
    internalThreshold: 0.35,      // 响应阈值 0.3-0.6
    randomExploreProb: 0.12,      // 随机探索概率 0.1-0.2

    // 统计数据
    stats: {
      pheromoneDeposits: 3,       // 信息素沉积次数
      explorationRounds: 2,       // 探索轮数
      findingsCount: 5,           // 发现数量
      signalsSent: 1,             // 发送的停止信号数
      signalsReceived: 2          // 接收的抑制信号数
    },

    // 当前状态
    current: {
      exploringDirection: "智能补货预测",
      claimedSubtask: "subtask-123",
      roleTransitionPending: false
    },

    // 角色历史
    roleHistory: [
      { from: "EXPLORER", to: "DEEP_ANALYST", reason: "...", timestamp: 1234567890 }
    ],

    // 状态
    status: "active" | "idle" | "degraded" | "terminated"
  }
}
```

#### Agent初始化/恢复流程

```javascript
// Orchestrator在每轮开始时发送
{
  type: "round_start",
  round: 3,
  agentState: {
    role: "EXPLORER",
    internalThreshold: 0.35,
    randomExploreProb: 0.12,
    stats: { pheromoneDeposits: 3, explorationRounds: 2, findingsCount: 5 },
    current: { exploringDirection: "智能补货预测" }
  },
  blackboardSnapshot: {
    pheromones: { ... },
    stopSignals: [ ... ],
    claims: { ... },
    findings: [ ... ]
  }
}
```

#### Agent更新状态

```javascript
// Agent在探索过程中更新
SendMessage({
  type: "blackboard_operation",
  operation: "update_agent_state",
  params: {
    updates: {
      "stats.pheromoneDeposits": 4,
      "stats.explorationRounds": 3,
      "current.exploringDirection": "农网智能化"
    }
  },
  summary: "更新Agent状态"
})
```

---

### 3.3 方案三：多轮迭代协议

#### 设计思路

设计明确的轮次机制，Agent持续参与多轮探索直到收敛。

#### 轮次生命周期

```
轮次流程：

┌─────────────────────────────────────────────────────────────┐
│ Round N                                                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Orchestrator广播 round_start                             │
│     ├── 包含：轮次号、Agent状态、黑板快照                    │
│     └── 所有Agent同时收到                                    │
│                                                              │
│  2. Agent执行探索                                            │
│     ├── 读取黑板状态                                         │
│     ├── 选择探索方向（基于信息素/随机）                      │
│     ├── 执行探索                                             │
│     ├── 沉积信息素                                           │
│     ├── 发送停止信号（如有冲突）                             │
│     └── 更新Agent状态                                        │
│                                                              │
│  3. Agent发送 round_complete                                 │
│     ├── 包含：本轮发现、信息素操作、状态更新                 │
│     └── Orchestrator收集所有响应                             │
│                                                              │
│  4. Orchestrator轮次结算                                     │
│     ├── 处理所有黑板操作请求                                 │
│     ├── 执行信息素蒸发                                       │
│     ├── 清理过期停止信号                                     │
│     ├── 记录观点历史                                         │
│     └── 检查收敛状态                                         │
│                                                              │
│  5. 收敛判断                                                 │
│     ├── 如果收敛 → 广播 swarm_converged → 结束              │
│     └── 如果未收敛 → 进入Round N+1                           │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 消息协议

```javascript
// Orchestrator → All: 轮次开始
{
  type: "round_start",
  round: 3,
  config: {
    roundTimeout: 120000,  // 2分钟
    operations: ["deposit_pheromone", "send_stop_signal", "update_finding"]
  },
  agentState: { /* Agent的完整状态 */ },
  blackboardSnapshot: {
    pheromones: { "智能补货": { concentration: 0.65, ... }, ... },
    stopSignals: [ ... ],
    findings: [ ... ],
    claims: { ... }
  }
}

// Agent → Orchestrator: 轮次完成
{
  type: "round_complete",
  round: 3,
  report: {
    direction: "智能补货预测",
    findings: [
      { coreIdea: "补货预测ROI最高", perspective: "商业价值", details: "..." }
    ],
    operations: [
      { operation: "deposit_pheromone", params: { direction: "智能补货", amount: 0.1 } },
      { operation: "update_agent_state", params: { updates: { "stats.pheromoneDeposits": 4 } } }
    ]
  }
}

// Orchestrator → All: 收敛达成
{
  type: "swarm_converged",
  convergence: {
    stable: true,
    quorumReached: true,
    consensusPoints: [ ... ],
    diversityScore: 0.65
  },
  finalReport: { /* 完整报告 */ }
}
```

#### Orchestrator轮次协调器

```javascript
class RoundCoordinator {
  constructor(blackboard, config) {
    this.blackboard = blackboard;
    this.config = {
      maxRounds: 10,
      roundTimeout: 120000,
      betaStability: 2,
      quorumThreshold: 0.67
    };
    this.currentRound = 0;
    this.pendingOperations = [];
    this.roundResponses = [];
  }

  async runSwarm() {
    while (this.currentRound < this.config.maxRounds) {
      this.currentRound++;

      // 1. 广播轮次开始
      await this.broadcastRoundStart();

      // 2. 等待Agent响应
      const responses = await this.waitForResponses();

      // 3. 处理操作
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
    const agents = Object.keys(this.blackboard.metadata.agentStates);

    for (let agentId of agents) {
      const agentState = this.blackboard.metadata.agentStates[agentId];

      await SendMessage({
        type: "message",
        recipient: agentId,
        content: JSON.stringify({
          type: "round_start",
          round: this.currentRound,
          agentState: agentState,
          blackboardSnapshot: this.getSnapshot()
        }),
        summary: `Round ${this.currentRound} 开始`
      });
    }
  }

  async waitForResponses() {
    const agents = Object.keys(this.blackboard.metadata.agentStates);
    const responses = [];

    // 并行等待所有Agent响应
    const promises = agents.map(agentId =>
      this.waitForAgentResponse(agentId, this.config.roundTimeout)
    );

    const results = await Promise.allSettled(promises);

    for (let result of results) {
      if (result.status === 'fulfilled' && result.value) {
        responses.push(result.value);
      }
    }

    return responses;
  }

  async processOperations(responses) {
    for (let response of responses) {
      for (let op of response.report.operations) {
        await this.executeOperation(op.operation, op.params, response.agentId);
      }
    }
  }

  async settleRound() {
    // 1. 信息素蒸发
    this.evaporatePheromones();

    // 2. 清理过期停止信号
    this.cleanExpiredSignals();

    // 3. 记录观点历史
    this.recordOpinionHistory();

    // 4. 更新Agent轮次计数
    for (let agentId in this.blackboard.metadata.agentStates) {
      this.blackboard.metadata.agentStates[agentId].stats.explorationRounds++;
    }
  }

  evaporatePheromones() {
    const ph = this.blackboard.metadata.pheromones;
    const rate = this.blackboard.metadata.config.evaporationRate || 0.05;

    for (let direction in ph) {
      ph[direction].concentration *= (1 - rate);
      ph[direction].lastUpdate = Date.now();

      if (ph[direction].concentration < 0.1) {
        delete ph[direction];
      }
    }
  }

  checkConvergence() {
    // β稳定性检查
    const history = this.blackboard.metadata.opinionHistory;
    if (history.length < this.config.betaStability) {
      return { converged: false, reason: "insufficient_history" };
    }

    const recent = history.slice(-this.config.betaStability);
    const opinionSets = recent.map(r =>
      new Set(r.findings.map(f => f.coreIdea))
    );

    const isStable = opinionSets.every(set =>
      setsEqual(set, opinionSets[0])
    );

    if (!isStable) {
      return { converged: false, reason: "not_stable" };
    }

    // 法定人数检查
    const latestFindings = history[history.length - 1].findings;
    const agentCount = Object.keys(this.blackboard.metadata.agentStates).length;

    const supportCount = {};
    for (let f of latestFindings) {
      supportCount[f.coreIdea] = (supportCount[f.coreIdea] || 0) + 1;
    }

    const consensusPoints = Object.entries(supportCount)
      .filter(([_, count]) => count / agentCount >= this.config.quorumThreshold)
      .map(([idea, count]) => ({ idea, support: count, rate: count / agentCount }));

    return {
      converged: isStable && consensusPoints.length > 0,
      stable: isStable,
      consensusPoints: consensusPoints,
      diversity: this.calculateDiversity()
    };
  }
}
```

---

## 四、P1方案详解

### 4.1 方案四：超时降级机制

#### 设计思路

当Agent在指定时间内未响应时，自动降级处理，保证协作继续进行。

#### 超时处理流程

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
│              ▼                                               │
│       记录超时事件                                           │
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

#### 实现代码

```javascript
class AgentTimeoutHandler {
  constructor(config) {
    this.config = {
      responseTimeout: 60000,      // 1分钟
      maxRetries: 2,
      minActiveAgents: 2           // 最少活跃Agent数
    };
    this.agentStatus = {};
  }

  async waitForAgentResponse(agentId, timeout) {
    return new Promise((resolve) => {
      const timer = setTimeout(() => {
        this.handleTimeout(agentId);
        resolve(null);
      }, timeout);

      // 监听响应
      this.onMessage(agentId, "round_complete", (msg) => {
        clearTimeout(timer);
        this.resetTimeout(agentId);
        resolve(msg);
      });
    });
  }

  handleTimeout(agentId) {
    const status = this.agentStatus[agentId] || { timeoutCount: 0 };
    status.timeoutCount++;
    status.lastTimeout = Date.now();
    this.agentStatus[agentId] = status;

    console.log(`Agent ${agentId} timeout (${status.timeoutCount}/${this.config.maxRetries})`);

    if (status.timeoutCount >= this.config.maxRetries) {
      this.degradeAgent(agentId);
    } else {
      this.retryAgent(agentId);
    }
  }

  degradeAgent(agentId) {
    // 更新状态
    this.blackboard.metadata.agentStates[agentId].status = "degraded";
    this.blackboard.metadata.agentStates[agentId].degradedAt = Date.now();

    // 从活跃列表移除
    this.activeAgents = this.activeAgents.filter(id => id !== agentId);

    // 通知Validator
    SendMessage({
      type: "message",
      recipient: "validator",
      content: JSON.stringify({
        type: "agent_degraded",
        agentId: agentId,
        activeCount: this.activeAgents.length,
        canContinue: this.canContinue()
      }),
      summary: `Agent ${agentId} 降级`
    });
  }

  canContinue() {
    return this.activeAgents.length >= this.config.minActiveAgents;
  }

  retryAgent(agentId) {
    SendMessage({
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
}
```

---

## 五、P2方案详解

### 5.1 方案五：角色转换触发器

#### 角色转换规则

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
            reason: `满足转换条件`
          });
          break;  // 每轮只转换一次
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

### 5.2 方案六：StopSignal协议

#### 完整实现

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

// Orchestrator处理停止信号效果
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

// 阈值响应函数
function responseProbability(stimulus, threshold) {
  if (stimulus === 0) return 0;
  return (stimulus ** 2) / (stimulus ** 2 + threshold ** 2);
}
```

---

## 六、实施路线图

### 6.1 阶段划分

```
Phase 1: 核心基础设施 (3-5天)
├── Day 1: 黑板代理机制设计与实现
│   ├── 定义操作类型
│   ├── 实现Orchestrator处理逻辑
│   └── 编写Agent使用文档
│
├── Day 2: Agent状态托管
│   ├── 设计状态结构
│   ├── 实现状态恢复流程
│   └── 测试状态持久化
│
└── Day 3-5: 多轮迭代协议
    ├── RoundCoordinator实现
    ├── 消息协议定义
    └── 基础测试

Phase 2: 增强功能 (2-3天)
├── Day 6: 超时降级机制
│
└── Day 7-8: 信息素完整实现
    ├── 真实沉积/蒸发
    ├── 方向选择逻辑
    └── StopSignal处理

Phase 3: 高级特性 (2天)
├── Day 9: 角色转换触发器
│
└── Day 10: 集成测试与优化

Phase 4: 文档与发布 (1天)
├── 更新skill.md
├── 编写使用指南
└── 发布v2.1
```

### 6.2 验证清单

| 功能 | 测试用例 | 预期结果 |
|------|----------|----------|
| 黑板操作 | Agent发送deposit_pheromone | 黑板信息素浓度增加 |
| 状态恢复 | Agent重启后读取状态 | 状态完整恢复 |
| 多轮迭代 | 运行3轮协作 | 每轮正常完成 |
| 超时降级 | Agent故意不响应 | 超时后降级 |
| 角色转换 | 信息素>0.7 | 触发DEEP_ANALYST转换 |
| 停止信号 | 发送stop_signal | 目标信息素降低 |

---

## 七、预期效果

### 7.1 对比表

| 能力 | v2.0 | v2.1 |
|------|------|------|
| 信息素系统 | 口头声明 | ✅ 真实操作 |
| Agent状态 | 无持久化 | ✅ 黑板托管 |
| 迭代能力 | 单轮 | ✅ 多轮迭代 |
| 超时处理 | 无 | ✅ 自动降级 |
| 角色演化 | 固定 | ✅ 动态转换 |
| 交叉抑制 | 未实现 | ✅ StopSignal协议 |
| 收敛判断 | 主观 | ✅ 量化Aegean |

### 7.2 成功指标

```
v2.1目标：

可靠性:
├── Agent响应率 ≥ 95%
├── 超时降级成功率 100%
└── 协作完成率 ≥ 90%

功能性:
├── 信息素操作成功率 100%
├── 状态恢复准确率 100%
├── 角色转换触发准确率 ≥ 90%
└── 收敛判断准确率 ≥ 95%

性能:
├── 单轮耗时 ≤ 2分钟
├── 总协作时间 ≤ 10分钟
└── 黑板操作延迟 ≤ 1秒
```

---

## 八、风险与对策

| 风险 | 可能性 | 影响 | 对策 |
|------|:------:|:----:|------|
| Orchestrator负载过高 | 中 | 高 | 异步处理操作队列 |
| 消息丢失 | 低 | 高 | 增加重试机制 |
| 状态冲突 | 低 | 中 | 乐观锁+版本号 |
| Agent拒绝角色转换 | 中 | 低 | 允许拒绝，保持原角色 |

---

*Swarm v2.1 升级方案 | 2026*
