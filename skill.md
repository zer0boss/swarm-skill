# Swarm v2.4 - 多智能体蜂群协作系统

> 基于群体智能理论，实现黑板代理、状态托管、多轮迭代、量化共识

**版本: v2.4.0** | 发布日期: 2026-02-18

### v2.4 更新内容（强制协议执行 + 研究成果归档）

| 问题 | 修复 | 优先级 |
|------|------|--------|
| 黑板代理未执行 | ✅ Agent **必须**通过SendMessage发送操作，禁止在报告中声明 | P0 |
| 阈值响应未体现 | ✅ 强制决策报告，**必须**包含P(S,θ)计算过程 | P0 |
| 停止信号未激活 | ✅ 强制冲突审查，每轮**必须**提交审查报告 | P1 |
| 强制终止需手动 | ✅ 自动强制终止机制，无需手动修改配置 | P1 |
| 随机探索未执行 | ✅ 合规性检查器验证，违规标记处罚 | P1 |
| 研究成果无归档 | ✅ 独立运行目录 + 研究报告自动生成 | P2 |
| Agent违规无处理 | ✅ 违规积分机制，超阈值强制终止 | P1 |

### v2.4 新增功能

| 功能 | 说明 |
|------|------|
| **运行目录管理** | 每次运行创建独立目录 `swarm-runs/YYYY-MM-DD-task-slug/` |
| **研究报告生成** | 自动生成 `final-research-report.md` 人类可读报告 |
| **合规性检查器** | `AGENT_COMPLIANCE_CHECKER` 验证Agent行为是否符合协议 |
| **违规处理器** | `VIOLATION_HANDLER` 累计违规分数，触发降级/终止 |
| **自动强制终止** | `AUTO_FORCE_TERMINATE` 无需手动干预的强制终止机制 |

### v2.3 更新内容（执行层修复 - 基于实际运行复盘）

| 问题 | 修复 | 优先级 |
|------|------|--------|
| 黑板代理未执行 | ✅ 新增操作队列+确认机制，必须返回operation_result | P0 |
| 信息素系统虚假 | ✅ 信息素只能通过Agent的deposit_pheromone修改 | P0 |
| Shutdown第三阶段未执行 | ✅ 强制终止机制，必须标记Agent为terminated | P0 |
| 阈值响应模型未应用 | ✅ 决策时强制计算responseProbability | P1 |
| Agent沉默未处理 | ✅ 沉默检测+强制唤醒+降级机制 | P1 |
| 收敛判断无计算 | ✅ CONVERGENCE_CALCULATOR输出具体数值 | P1 |
| Orchestrator跳过步骤 | ✅ 执行状态机，每步必须显式确认 | P0 |

### v2.2 更新内容（理论符合性修复）

| 问题 | 修复 | 优先级 |
|------|------|--------|
| Agent状态同步失效 | ✅ 新增状态同步协议，任务完成时强制更新 | P1 |
| Shutdown流程不完整 | ✅ 新增三阶段关闭协议（预通知→优雅关闭→强制终止） | P1 |
| 停止信号无效 | ✅ 信号立即降低目标信息素浓度，Agent必须响应 | P1 |
| 随机探索未实现 | ✅ 强制15%随机探索，在round_start中明确指令 | P2 |
| 角色转换未执行 | ✅ 轮次结算时强制检查并执行转换，立即通知Agent | P2 |
| 多样性保护缺失 | ✅ 最少3轮收敛，90%共识率上限，强制保护机制 | P2 |

### v2.1.2 更新内容

| 问题 | 修复 |
|------|------|
| Web搜索卡住 | ✅ 新增搜索超时与降级机制 |
| interrupted误判 | ✅ 新增Agent状态恢复机制 |
| Agent空闲浪费 | ✅ 新增空闲碰撞机制（碰撞协议） |
| 轮次中断恢复 | ✅ 新增轮次恢复机制 |

### v2.1.1 更新内容

| 问题 | 修复 |
|------|------|
| TeamDelete失败 | ✅ 新增 `force_cleanup` 强制清理机制 |
| 黑板代理未执行 | ✅ 新增 Orchestrator 操作执行清单 |
| 多轮迭代未验证 | ✅ 强制最少2轮（minRounds=2）|
| Agent响应时序 | ✅ 新增同步屏障 `sync_barrier` |
| shutdown延迟 | ✅ 新增 `shutdown_timeout` 配置 |

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
      // v2.2 更新配置
      evaporationRate: 0.08,      // 提高到8%（原5%），保持信息素梯度
      depositAmount: 0.1,
      maxAgentsPerTask: 3,
      betaStability: 2,
      quorumThreshold: 0.67,
      minDiversity: 0.4,
      maxRounds: 10,
      minRounds: 3,               // v2.2: 提高到3（原2），保证探索充分
      minRoundsBeforeConvergence: 3,  // v2.2: 最少3轮才能收敛
      maxConsensusRate: 0.9,      // v2.2: 单轮最大共识率90%
      roundTimeout: 120000,       // 2分钟
      responseTimeout: 60000,     // 1分钟
      syncBarrierTimeout: 60000,  // 同步屏障超时

      // v2.2 三阶段关闭配置
      preNotifyTimeout: 5000,     // 预通知: 5秒
      gracefulTimeout: 15000,     // 优雅关闭: 15秒
      forceCleanupTimeout: 10000, // 强制终止: 10秒

      forceCleanup: true          // 超时后强制清理
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

## v2.2 轮次指令（必须遵守）

收到 `round_start` 消息后，检查 `instructions` 字段：

### 情况1: forceRandomExplore = true
你**必须**忽略信息素，随机选择一个探索方向。
这是为了保证系统多样性，防止过早收敛。**这是强制性指令！**

### 情况2: mustSwitchDirection = true
你当前的方向被停止信号抑制，**必须**切换到其他方向。
- 你的阈值 = {internalThreshold}
- 有效浓度 = {effectiveConc}
- 当 effectiveConc < internalThreshold 时，必须切换

### 情况3: currentDirectionInhibited = true
你当前方向收到了停止信号，但还可以继续。
- 建议考虑其他方向
- 如果继续，需要提供更强的理由

### 情况4: 正常模式
按照信息素浓度选择方向。

## 停止信号处理（v2.2 强化）

当你发现矛盾证据时，发送停止信号：
```
SendMessage({
  type: "blackboard_operation",
  operation: "send_stop_signal",
  params: {
    targetDirection: "目标方向",
    reason: "contradictory_evidence",
    evidence: "证据描述"
  }
})
```

**重要**: 停止信号会立即降低目标方向的信息素浓度30%。
被抑制的Agent必须根据其阈值决定是否切换方向。

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

## 角色演化（v2.2 强制执行）

**重要变更**: 角色转换不再是"可能"，而是满足条件后**强制执行**。
你会在轮次结算时收到 `role_transition_executed` 消息。

### 转换条件（自动检测）
- 信息素>0.7 + 沉积>3次 → **转为** DeepAnalyst
- 发现冲突（发送停止信号）→ **转为** Debater
- 完成2轮探索 → **转为** Synthesizer

### 收到转换消息后
```
{
  "type": "role_transition_executed",
  "fromRole": "EXPLORER",
  "toRole": "DEEP_ANALYST",
  "capabilities": { ... },
  "instructions": "你的新角色指令..."
}
```

**必须**按照新角色的指令行动！

### 各角色职责

**DeepAnalyst**: 深入分析最高信息素浓度的方向，寻找深层逻辑和数据支持

**Debater**: 审查主流观点，寻找逻辑漏洞，发送停止信号挑战不充分的方向

**Synthesizer**: 整合所有发现，识别共性和分歧，生成阶段性综合报告

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

## 九、v2.3 核心协议（执行层修复）

### 9.0 v2.3 新增协议概述

| 协议 | 解决问题 | 核心机制 |
|------|----------|----------|
| **执行状态机** | Orchestrator跳过步骤 | 每步必须markStepCompleted/assertStepCompleted |
| **黑板操作队列** | 操作请求无确认 | pending→processed必须配对，返回operation_result |
| **信息素真实化** | 浓度手动设置 | 只能通过deposit_pheromone修改 |
| **强制终止机制** | Agent不关闭 | 第三阶段必须执行，标记terminated |
| **沉默检测恢复** | Agent无输出 | 检测沉默→强制唤醒→降级 |
| **收敛计算器** | 无实际计算 | 输出β稳定性、法定人数、多样性具体数值 |

### 9.0.1 Orchestrator执行状态机（v2.3核心）

```javascript
// v2.3 新增: 执行状态机
const ORCHESTRATOR_EXECUTION_STATE = {
  currentPhase: "INIT",
  completedSteps: new Set(),
  requiredSteps: {
    "INIT": ["create_team", "create_blackboard", "init_agent_states", "spawn_agents"],
    "ROUND": ["broadcast_round_start", "wait_responses", "process_operations",
              "return_results", "update_blackboard", "settle_round"],
    "CONVERGENCE": ["check_beta_stability", "check_quorum", "check_diversity", "generate_report"],
    "SHUTDOWN": ["pre_notify", "graceful_request", "force_terminate", "mark_all_terminated"]
  }
};

// 强制检查函数
function assertStepCompleted(phase, step) {
  const key = `${phase}:${step}`;
  if (!ORCHESTRATOR_EXECUTION_STATE.completedSteps.has(key)) {
    const error = `CRITICAL: Step ${key} not completed!`;
    console.error(`[STATE MACHINE] ${error}`);
    throw new Error(error);
  }
}

// 标记完成
function markStepCompleted(phase, step) {
  const key = `${phase}:${step}`;
  ORCHESTRATOR_EXECUTION_STATE.completedSteps.add(key);
  console.log(`[STATE MACHINE] ✓ ${key}`);
}

// 阶段完成验证
function assertPhaseCompleted(phase) {
  const required = ORCHESTRATOR_EXECUTION_STATE.requiredSteps[phase] || [];
  for (let step of required) {
    assertStepCompleted(phase, step);
  }
  console.log(`[STATE MACHINE] ✓✓ Phase ${phase} completed`);
}
```

### 9.0.2 黑板操作队列与确认机制（v2.3核心）

```javascript
// v2.3 新增: 黑板操作队列
const BLACKBOARD_OPERATION_QUEUE = {
  pending: [],
  processed: [],
  failed: []
};

// 新增: 操作处理函数（必须执行并返回结果）
async function processBlackboardOperation(message, blackboard, blackboardTaskId) {
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
  console.log(`[BLACKBOARD] Received: ${operation} from ${agentId}`);

  // 2. 执行操作
  const opDef = BLACKBOARD_OPERATIONS[operation];
  if (!opDef) {
    opRecord.status = "failed";
    opRecord.error = "unknown_operation";
    BLACKBOARD_OPERATION_QUEUE.failed.push(opRecord);

    // 必须返回失败结果
    await SendMessage({
      type: "message",
      recipient: agentId,
      content: JSON.stringify({
        type: "operation_result",
        operationId: opRecord.id,
        success: false,
        error: "unknown_operation"
      }),
      summary: `操作失败: 未知操作 ${operation}`
    });
    return { success: false, error: "unknown_operation" };
  }

  // 3. 执行handler
  const result = opDef.handler(blackboard, params, agentId);
  opRecord.status = "completed";
  opRecord.result = result;
  BLACKBOARD_OPERATION_QUEUE.processed.push(opRecord);

  // 4. 【关键】必须返回operation_result给Agent
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
  console.log(`[BLACKBOARD] Result sent to ${agentId}: ${result.success}`);

  // 5. 更新黑板Task
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

// 验证操作队列
function validateOperationQueue() {
  const pending = BLACKBOARD_OPERATION_QUEUE.pending.length;
  const processed = BLACKBOARD_OPERATION_QUEUE.processed.length;
  const failed = BLACKBOARD_OPERATION_QUEUE.failed.length;

  if (pending !== processed + failed) {
    console.error(`[VALIDATION] Operation queue mismatch: ${pending} pending, ${processed} processed, ${failed} failed`);
    return false;
  }
  console.log(`[VALIDATION] ✓ Operation queue: ${processed}/${pending} processed`);
  return true;
}
```

### 9.0.3 信息素真实化（v2.3核心）

```javascript
// v2.3: 信息素只能通过Agent的deposit_pheromone修改
// 禁止直接设置浓度！

// 信息素操作（唯一入口）
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

      // 记录沉积历史
      ph[direction].depositHistory = ph[direction].depositHistory || [];
      ph[direction].depositHistory.push({
        agentId,
        amount,
        timestamp: Date.now()
      });

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
  // ... 其他操作保持不变
};

// 信息素蒸发（轮次结算时自动执行）
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

// ⚠️ 禁止行为
// ❌ blackboard.metadata.pheromones["方向A"].concentration = 0.8;
// ✅ 必须通过Agent的deposit_pheromone操作
```

### 9.0.4 Shutdown强制终止机制（v2.3核心）

```javascript
// v2.3: 第三阶段必须执行
const FORCE_TERMINATE_PROTOCOL = {
  // 阶段3: 强制终止
  execute: async (agentId, blackboard, teamName) => {
    console.warn(`[FORCE TERMINATE] Agent ${agentId} 强制终止`);

    // 1. 标记Agent状态
    if (blackboard.metadata.agentStates[agentId]) {
      blackboard.metadata.agentStates[agentId].status = "terminated";
      blackboard.metadata.agentStates[agentId].terminatedAt = Date.now();
      blackboard.metadata.agentStates[agentId].terminationReason = "forced";
      blackboard.metadata.agentStates[agentId].terminationRound = blackboard.metadata.currentRound;
    }

    // 2. 记录终止事件
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
async function executeShutdown(agents, blackboard, blackboardTaskId, teamName) {
  const results = { graceful: [], forced: [], failed: [] };

  // 阶段1: 预通知（5秒）
  console.log("[SHUTDOWN] Phase 1: Pre-notify");
  markStepCompleted("SHUTDOWN", "pre_notify_started");
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
  markStepCompleted("SHUTDOWN", "graceful_request_started");
  for (let agent of agents) {
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
        blackboard.metadata.agentStates[agent.id].status = "terminated";
        blackboard.metadata.agentStates[agent.id].terminationReason = "graceful";
        console.log(`[SHUTDOWN] ✓ ${agent.id} 优雅关闭`);
      }
    } catch (e) {
      console.log(`[SHUTDOWN] ✗ ${agent.id} 无响应: ${e.message}`);
    }
  }
  markStepCompleted("SHUTDOWN", "graceful_request");

  // 阶段3: 强制终止（必须执行！）
  console.log("[SHUTDOWN] Phase 3: Force terminate");
  const remainingAgents = agents.filter(a =>
    !results.graceful.includes(a.id) &&
    blackboard.metadata.agentStates[a.id]?.status !== "terminated"
  );

  if (remainingAgents.length > 0) {
    console.warn(`[SHUTDOWN] ⚠️ ${remainingAgents.length} 个Agent需要强制终止`);

    for (let agent of remainingAgents) {
      await FORCE_TERMINATE_PROTOCOL.execute(agent.id, blackboard, teamName);
      results.forced.push(agent.id);
      console.log(`[SHUTDOWN] ⚡ ${agent.id} 强制终止`);
    }
  }
  markStepCompleted("SHUTDOWN", "force_terminate");

  // 验证所有Agent已终止
  const allTerminated = Object.values(blackboard.metadata.agentStates)
    .every(s => s.status === "terminated");
  if (!allTerminated) {
    console.error("[SHUTDOWN] ❌ 仍有Agent未终止!");
  }
  markStepCompleted("SHUTDOWN", "mark_all_terminated");

  // 更新黑板
  await TaskUpdate({
    taskId: blackboardTaskId,
    metadata: {
      ...blackboard.metadata,
      shutdownResult: results
    }
  });

  console.log(`[SHUTDOWN] 完成: ${results.graceful.length} 优雅, ${results.forced.length} 强制`);
  return results;
}
```

### 9.0.5 Agent沉默检测与恢复（v2.3新增）

```javascript
// v2.3 新增: Agent沉默检测
const AGENT_SILENCE_DETECTION = {
  maxSilenceTime: 60000,     // 60秒无响应视为沉默
  maxSilenceRounds: 2,       // 连续2轮沉默触发恢复

  // 检测沉默
  detectSilence: (blackboard, agentId) => {
    const state = blackboard.metadata.agentStates[agentId];
    if (!state) return { silent: false };

    const lastActive = state.lastActiveAt || 0;
    const silenceTime = Date.now() - lastActive;
    const silenceRounds = state.stats?.silenceRounds || 0;

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

### 9.0.6 收敛计算器（v2.3新增）

```javascript
// v2.3 新增: 收敛计算器 - 必须输出具体数值
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
    const isStable = opinionSets.every(set => setsEqual(set, firstSet));

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
      reason: "",
      calculationDetails: {}  // 必须包含计算细节
    };

    // 1. 最少轮数
    if (!result.minRoundsMet) {
      result.reason = `轮数不足: ${currentRound}/${minRounds}`;
      return result;
    }

    // 2. β稳定性
    result.betaStability = CONVERGENCE_CALCULATOR.checkBetaStability(blackboard, betaStability);
    result.calculationDetails.betaStability = result.betaStability;
    if (!result.betaStability.stable) {
      result.reason = `观点不稳定: ${result.betaStability.reason}`;
      return result;
    }

    // 3. 法定人数
    result.quorum = CONVERGENCE_CALCULATOR.checkQuorum(blackboard, quorumThreshold);
    result.calculationDetails.quorum = result.quorum;
    if (!result.quorum.quorum) {
      result.reason = `无法定人数共识`;
      return result;
    }

    // 4. 多样性
    result.diversity = CONVERGENCE_CALCULATOR.checkDiversity(blackboard, minDiversity);
    result.calculationDetails.diversity = result.diversity;
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

### 9.0.7 阈值响应模型强制应用（v2.3强化）

```javascript
// 阈值响应函数（理论核心）
function responseProbability(stimulus, threshold) {
  if (stimulus === 0) return 0;
  // P(S,θ) = S² / (S² + θ²)
  return (stimulus ** 2) / (stimulus ** 2 + threshold ** 2);
}

// v2.3: Agent决策辅助函数（强制计算）
function calculateAgentDecision(blackboard, agentId, candidateDirections) {
  const state = blackboard.metadata.agentStates[agentId];
  const threshold = state.internalThreshold;

  const decisions = candidateDirections.map(dir => {
    const rawConc = blackboard.metadata.pheromones[dir]?.concentration || 0;

    // 检查停止信号抑制
    const inhibition = checkInhibition(blackboard, agentId, dir);
    const effectiveConc = inhibition.effectiveConcentration || rawConc;

    // 【关键】计算响应概率
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

    // 【v2.3关键】计算阈值响应决策
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
        // 【v2.3新增】决策辅助信息
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
  markStepCompleted("ROUND", "broadcast_round_start");
}
```

### 9.0.8 状态同步协议

```javascript
// 任务完成时必须执行
const STATE_SYNC_PROTOCOL = {
  onComplete: async (blackboard, convergenceResult) => {
    // 1. 标记黑板任务完成
    await TaskUpdate({
      taskId: blackboardTaskId,
      status: "completed",
      metadata: {
        ...blackboard.metadata,
        convergenceResult: convergenceResult,
        completedAt: Date.now()
      }
    });

    // 2. 更新所有Agent状态
    for (let agentId in blackboard.metadata.agentStates) {
      blackboard.metadata.agentStates[agentId].status = "completed";
    }

    // 3. 记录完成事件
    blackboard.metadata.events = blackboard.metadata.events || [];
    blackboard.metadata.events.push({
      type: "swarm_completed",
      round: currentRound,
      convergenceResult: convergenceResult,
      timestamp: Date.now()
    });

    return { synced: true };
  }
};
```

### 9.0.2 三阶段关闭协议

```javascript
const SHUTDOWN_PROTOCOL = {
  phases: {
    // 阶段1: 预通知（5秒）
    preNotify: { timeout: 5000, action: "broadcast_shutdown_imminent" },

    // 阶段2: 正式请求（15秒）
    gracefulRequest: { timeout: 15000, action: "send_shutdown_request" },

    // 阶段3: 强制终止（10秒后）
    forceTerminate: { timeout: 10000, action: "force_cleanup" }
  },

  execute: async (agents, blackboard) => {
    const results = { graceful: [], forced: [], failed: [] };

    // 阶段1: 预通知
    await broadcast({ type: "shutdown_imminent", prepareTime: 5000 });
    await sleep(5000);

    // 阶段2: 正式请求
    for (let agent of agents) {
      const result = await sendShutdownRequest(agent.id, 15000);
      if (result.acknowledged) {
        results.graceful.push(agent.id);
      } else {
        results.failed.push(agent.id);
      }
    }

    // 阶段3: 强制清理
    const remaining = agents.filter(a => !results.graceful.includes(a.id));
    if (remaining.length > 0) {
      for (let agent of remaining) {
        blackboard.metadata.agentStates[agent.id].status = "terminated";
        blackboard.metadata.agentStates[agent.id].terminationReason = "forced";
        results.forced.push(agent.id);
      }
    }

    return { success: true, ...results };
  }
};
```

### 9.0.3 停止信号强化（必须执行）

```javascript
// 发送停止信号时，立即降低目标信息素
const ENHANCED_STOP_SIGNAL = {
  handler: (blackboard, params, agentId) => {
    const signal = {
      id: `signal-${Date.now()}`,
      from: agentId,
      target: params.targetDirection,
      reason: params.reason,
      evidence: params.evidence,
      strength: 0.3,
      timestamp: Date.now(),
      active: true
    };

    blackboard.metadata.stopSignals.push(signal);

    // 【关键】立即降低目标信息素浓度
    const targetPh = blackboard.metadata.pheromones[params.targetDirection];
    if (targetPh) {
      targetPh.concentration *= (1 - signal.strength);
      targetPh.suppressedBy = targetPh.suppressedBy || [];
      targetPh.suppressedBy.push(signal.id);
    }

    blackboard.metadata.agentStates[agentId].stats.signalsSent++;

    return {
      success: true,
      signalId: signal.id,
      suppressedConcentration: targetPh?.concentration
    };
  },

  // Agent选择方向时检查抑制
  checkInhibition: (blackboard, agentId, direction) => {
    const signals = blackboard.metadata.stopSignals.filter(
      s => s.target === direction && s.active
    );

    if (signals.length === 0) return { inhibited: false };

    const totalStrength = Math.min(
      signals.reduce((sum, s) => sum + s.strength, 0),
      0.5
    );

    const rawConc = blackboard.metadata.pheromones[direction]?.concentration || 0;
    const effectiveConc = rawConc * (1 - totalStrength);

    const state = blackboard.metadata.agentStates[agentId];
    const responseProb = (effectiveConc ** 2) / (effectiveConc ** 2 + state.internalThreshold ** 2);

    return {
      inhibited: Math.random() > responseProb,
      shouldSwitch: effectiveConc < state.internalThreshold,
      totalStrength, rawConc, effectiveConc
    };
  }
};
```

### 9.0.4 随机探索强制执行

```javascript
// 在 broadcastRoundStart 中强制执行
async function broadcastRoundStart(round, blackboard) {
  for (let agentId in blackboard.metadata.agentStates) {
    const state = blackboard.metadata.agentStates[agentId];
    if (state.status !== "active") continue;

    // 【关键】计算是否强制随机探索
    const forceRandomExplore = Math.random() < state.randomExploreProb;

    // 检查抑制状态
    const currentDir = state.current.exploringDirection;
    const inhibition = currentDir
      ? ENHANCED_STOP_SIGNAL.checkInhibition(blackboard, agentId, currentDir)
      : { inhibited: false };

    await SendMessage({
      type: "message",
      recipient: agentId,
      content: JSON.stringify({
        type: "round_start",
        round: round,
        agentState: state,
        blackboardSnapshot: { /* ... */ },
        // 【新增】强制指令
        instructions: {
          forceRandomExplore: forceRandomExplore,
          mustSwitchDirection: inhibition.shouldSwitch,
          currentDirectionInhibited: inhibition.inhibited
        }
      }),
      summary: `Round ${round} 开始 ${forceRandomExplore ? '(强制随机探索)' : ''}`
    });
  }
}
```

### 9.0.5 角色转换强制执行

```javascript
// 在 settleRound 中强制执行
async function settleRound(blackboard) {
  const transitions = checkRoleTransitions(blackboard);

  for (let transition of transitions) {
    const state = blackboard.metadata.agentStates[transition.agentId];

    // 执行转换
    const oldRole = state.role;
    state.role = transition.toRole;
    state.roleHistory = state.roleHistory || [];
    state.roleHistory.push({
      from: oldRole, to: transition.toRole,
      reason: transition.reason, timestamp: Date.now()
    });

    // 【关键】立即通知Agent
    await SendMessage({
      type: "message",
      recipient: transition.agentId,
      content: JSON.stringify({
        type: "role_transition_executed",
        fromRole: oldRole,
        toRole: transition.toRole,
        reason: transition.reason,
        capabilities: ROLE_CAPABILITIES[transition.toRole],
        instructions: getRoleInstructions(transition.toRole)
      }),
      summary: `角色转换: ${oldRole} → ${transition.toRole}`
    });
  }
}

const ROLE_CAPABILITIES = {
  DEEP_ANALYST: {
    description: "深入分析高价值方向",
    canDo: ["deep_dive", "strengthen_pheromone"],
    focusOn: "最高信息素浓度的方向"
  },
  DEBATER: {
    description: "挑战主流观点",
    canDo: ["send_stop_signal", "propose_alternative"],
    focusOn: "寻找主流观点的漏洞"
  },
  SYNTHESIZER: {
    description: "整合多个观点",
    canDo: ["merge_findings", "generate_summary"],
    focusOn: "综合所有发现"
  }
};
```

### 9.0.6 多样性保护机制

```javascript
const DIVERSITY_PROTECTION = {
  minRoundsBeforeConvergence: 3,
  minDiversityThreshold: 0.4,
  maxConsensusRate: 0.9,

  canConverge: (blackboard, currentRound) => {
    // 1. 最少轮数检查
    if (currentRound < 3) {
      return { allowed: false, reason: `min_rounds: ${currentRound}/3` };
    }

    // 2. 多样性检查
    const diversity = calculateDiversity(blackboard);
    if (diversity.overall < 0.4) {
      return { allowed: false, reason: `diversity: ${diversity.overall.toFixed(2)}/0.4` };
    }

    // 3. 共识率检查
    const consensusRate = calculateConsensusRate(blackboard);
    if (consensusRate > 0.9 && currentRound < 5) {
      return { allowed: false, reason: `consensus_too_fast: ${consensusRate}` };
    }

    return { allowed: true, diversity, consensusRate };
  },

  protect: async (blackboard) => {
    const actions = [];

    // 1. 强制随机探索
    for (let agentId in blackboard.metadata.agentStates) {
      blackboard.metadata.agentStates[agentId].forceRandomExplore = true;
    }
    actions.push("all_agents_force_random");

    // 2. 激活Debater
    const explorers = Object.entries(blackboard.metadata.agentStates)
      .filter(([_, s]) => s.role === "EXPLORER" && s.status === "active");
    if (explorers.length > 0) {
      explorers[0][1].role = "DEBATER";
      actions.push(`activated_debater:${explorers[0][0]}`);
    }

    // 3. 降低主导信息素
    const pheromones = blackboard.metadata.pheromones;
    const maxDir = Object.entries(pheromones)
      .sort((a, b) => b[1].concentration - a[1].concentration)[0];
    if (maxDir && maxDir[1].concentration > 0.7) {
      maxDir[1].concentration *= 0.7;
      actions.push(`suppressed:${maxDir[0]}`);
    }

    return { actions };
  }
};
```

---

## 十、自我约束规则

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
| v2.1 | 2026-02-17 | 黑板代理、状态托管、多轮迭代、超时降级、角色触发器、StopSignal协议、独特Agent命名 |
| v2.1.1 | 2026-02-18 | 强制清理机制、同步屏障、最少轮数限制、黑板操作执行清单、shutdown超时配置 |
| v2.1.2 | 2026-02-18 | Web搜索降级、Agent状态恢复、空闲碰撞机制、轮次恢复机制 |
| v2.2 | 2026-02-18 | 理论符合性修复：状态同步协议、三阶段关闭、停止信号强化、强制随机探索、角色转换执行、多样性保护 |
| **v2.3** | 2026-02-18 | **执行层修复**：执行状态机、黑板操作队列、信息素真实化、强制终止机制、沉默检测恢复、收敛计算器 |

### v2.3 重点改进（执行层修复）

| 问题 | 修复 |
|------|------|
| 黑板代理未执行 | ✅ 操作队列+确认机制，必须返回operation_result |
| 信息素虚假 | ✅ 只能通过Agent的deposit_pheromone修改 |
| Shutdown第三阶段未执行 | ✅ 强制终止，必须标记terminated |
| 阈值响应未应用 | ✅ 决策时强制计算responseProbability |
| Agent沉默 | ✅ 沉默检测+强制唤醒+降级 |
| 收敛无计算 | ✅ CONVERGENCE_CALCULATOR输出具体数值 |
| Orchestrator跳过步骤 | ✅ 执行状态机，每步必须确认 |

### v2.2 重点改进（理论符合性）

| 问题 | 修复 |
|------|------|
| Agent状态不同步 | ✅ 收敛时强制更新黑板和Agent状态为completed |
| Shutdown循环 | ✅ 三阶段关闭：预通知→优雅关闭→强制终止 |
| 停止信号无效 | ✅ 信号立即降低目标信息素30%，Agent必须响应 |
| 无随机探索 | ✅ round_start明确指令forceRandomExplore |
| 角色不演化 | ✅ 轮次结算强制检查并执行转换，立即通知Agent |
| 过早收敛 | ✅ 最少3轮收敛、90%共识率上限、强制保护动作 |

---

## 十二、v2.1.1 新增机制

### 12.1 强制清理机制 (force_cleanup)

解决 TeamDelete 失败问题：

```javascript
async function cleanupTeam(teamName, options = {}) {
  const config = {
    shutdownTimeout: 30000,    // 等待Agent关闭的最长时间
    forceAfterTimeout: true,   // 超时后强制删除
    retryAttempts: 3           // 重试次数
  };

  // 1. 发送shutdown请求给所有Agent
  const agents = getActiveAgents(teamName);
  for (let agent of agents) {
    await SendMessage({
      type: "shutdown_request",
      recipient: agent.id,
      content: "协作结束，请关闭"
    });
  }

  // 2. 等待所有Agent关闭
  const startTime = Date.now();
  while (Date.now() - startTime < config.shutdownTimeout) {
    const activeCount = getActiveAgents(teamName).length;
    if (activeCount === 0) {
      break;
    }
    await sleep(1000);
  }

  // 3. 如果超时，强制删除团队目录
  const remainingAgents = getActiveAgents(teamName);
  if (remainingAgents.length > 0 && config.forceAfterTimeout) {
    console.warn(`强制清理 ${remainingAgents.length} 个未响应Agent`);
    await Bash({
      command: `rm -rf ~/.claude/teams/${teamName}/`
    });
  }

  return { success: true, forcedCleanup: remainingAgents.length > 0 };
}
```

### 12.2 同步屏障 (sync_barrier)

解决Agent响应时序不一致问题：

```javascript
class SyncBarrier {
  constructor(agentCount, timeout = 60000) {
    this.agentCount = agentCount;
    this.timeout = timeout;
    this.responses = new Map();
    this.barrierPromise = null;
    this.resolveBarrier = null;
  }

  async waitForAll(agentId, response) {
    // 记录响应
    this.responses.set(agentId, {
      response,
      timestamp: Date.now()
    });

    // 如果所有Agent都已响应，解除屏障
    if (this.responses.size >= this.agentCount) {
      if (this.resolveBarrier) {
        this.resolveBarrier(this.getAllResponses());
      }
      return { synced: true, allResponded: true };
    }

    // 否则等待
    return new Promise((resolve) => {
      const timeoutId = setTimeout(() => {
        resolve({
          synced: true,
          allResponded: false,
          respondedCount: this.responses.size,
          expectedCount: this.agentCount
        });
      }, this.timeout);

      // 设置屏障解除回调
      this.resolveBarrier = (responses) => {
        clearTimeout(timeoutId);
        resolve({ synced: true, allResponded: true, responses });
      };
    });
  }

  getAllResponses() {
    return Object.fromEntries(this.responses);
  }
}

// 使用示例
const barrier = new SyncBarrier(4, 60000);  // 4个Agent，60秒超时

// 在收到每个Agent响应时
agentResponses.forEach(async (response, agentId) => {
  const result = await barrier.waitForAll(agentId, response);
  if (result.allResponded) {
    console.log("所有Agent已同步，进入下一轮");
    await settleRound();
  } else {
    console.log(`超时: ${result.respondedCount}/${result.expectedCount} Agent响应`);
    await handleTimeout();
  }
});
```

### 12.3 最少轮数限制 (minRounds)

强制验证β稳定性：

```javascript
const SWARM_CONFIG = {
  // ... 其他配置
  minRounds: 2,              // 最少运行轮数（验证β稳定性需要）
  betaStability: 2,          // β稳定性检查窗口
  maxRounds: 10              // 最大轮数
};

async function checkConvergence(blackboard, currentRound) {
  // 强制最少轮数
  if (currentRound < SWARM_CONFIG.minRounds) {
    return {
      converged: false,
      reason: `min_rounds_not_reached: ${currentRound}/${SWARM_CONFIG.minRounds}`
    };
  }

  // β稳定性检查
  const history = blackboard.metadata.opinionHistory;
  if (history.length < SWARM_CONFIG.betaStability) {
    return {
      converged: false,
      reason: `insufficient_history: ${history.length}/${SWARM_CONFIG.betaStability}`
    };
  }

  // 检查最近β轮的观点是否稳定
  const recent = history.slice(-SWARM_CONFIG.betaStability);
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
  // ...

  return { converged: true };
}
```

### 12.4 黑板操作执行清单

Orchestrator必须执行的完整操作列表：

```javascript
// ===== Orchestrator 执行清单 =====
// 收到Agent的 blackboard_operation 消息后，必须执行以下步骤：

async function executeBlackboardOperation(message, blackboard) {
  const { operation, params } = message;
  const agentId = message.from;

  // 1. 验证操作类型
  const opDef = BLACKBOARD_OPERATIONS[operation];
  if (!opDef) {
    return { success: false, error: "unknown_operation" };
  }

  // 2. 验证参数
  const validation = validateParams(params, opDef.params);
  if (!validation.valid) {
    return { success: false, error: "invalid_params", details: validation.errors };
  }

  // 3. 执行操作
  const result = opDef.handler(blackboard, params, agentId);

  // 4. 更新黑板metadata（如果操作成功）
  if (result.success) {
    // 黑板已通过handler更新
    // 记录操作日志
    blackboard.metadata.operationLog = blackboard.metadata.operationLog || [];
    blackboard.metadata.operationLog.push({
      operation,
      agentId,
      params,
      result,
      timestamp: Date.now()
    });
  }

  // 5. 返回结果给Agent
  return {
    success: result.success,
    operationId: `op-${Date.now()}`,
    ...result
  };
}

// ===== 实际执行时Orchestrator的操作 =====

// 当收到 Agent 发送的消息：
// { type: "blackboard_operation", operation: "deposit_pheromone", params: { direction: "xxx", amount: 0.1 } }

// Orchestrator 执行：
// 1. 解析消息
// 2. 调用 executeBlackboardOperation()
// 3. 使用 TaskUpdate 更新黑板 Task 的 metadata
// 4. 使用 SendMessage 返回 operation_result 给 Agent

// 示例代码：
async function handleAgentMessage(message) {
  if (message.type === "blackboard_operation") {
    const result = await executeBlackboardOperation(message, currentBlackboard);

    // 更新黑板
    await TaskUpdate({
      taskId: blackboardTaskId,
      metadata: {
        pheromones: currentBlackboard.metadata.pheromones,
        findings: currentBlackboard.metadata.findings,
        stopSignals: currentBlackboard.metadata.stopSignals,
        agentStates: currentBlackboard.metadata.agentStates,
        operationLog: currentBlackboard.metadata.operationLog
      }
    });

    // 返回结果
    await SendMessage({
      type: "message",
      recipient: message.from,
      content: JSON.stringify({
        type: "operation_result",
        ...result
      }),
      summary: `黑板操作: ${message.operation} - ${result.success ? '成功' : '失败'}`
    });
  }
}
```

### 12.5 shutdown超时配置

```javascript
const SHUTDOWN_CONFIG = {
  gracefulTimeout: 10000,    // 优雅关闭等待时间: 10秒
  forceKillTimeout: 30000,   // 强制终止等待时间: 30秒
  retryInterval: 2000,       // 重试间隔: 2秒
  maxRetries: 3              // 最大重试次数
};

async function gracefulShutdown(agents) {
  for (let attempt = 1; attempt <= SHUTDOWN_CONFIG.maxRetries; attempt++) {
    // 发送shutdown请求
    for (let agent of agents) {
      await SendMessage({
        type: "shutdown_request",
        recipient: agent.id,
        content: `请关闭 (尝试 ${attempt}/${SHUTDOWN_CONFIG.maxRetries})`
      });
    }

    // 等待响应
    await sleep(SHUTDOWN_CONFIG.gracefulTimeout);

    // 检查是否全部关闭
    const activeAgents = agents.filter(a => a.status !== "terminated");
    if (activeAgents.length === 0) {
      return { success: true, method: "graceful" };
    }

    console.log(`仍有 ${activeAgents.length} 个Agent未关闭，重试...`);
  }

  // 超时后强制清理
  return { success: true, method: "forced" };
}
```

---

## 十三、Orchestrator 执行检查清单

每次运行Swarm时，Orchestrator必须完成以下步骤：

### 初始化阶段
- [ ] TeamCreate 创建团队
- [ ] TaskCreate 创建黑板（不设置owner）
- [ ] 初始化 agentStates（包含内在阈值）
- [ ] 启动所有Explorer Agent

### 每轮执行
- [ ] 广播 round_start（包含agentState和黑板快照）
- [ ] 等待所有Agent响应（使用同步屏障）
- [ ] 处理所有 blackboard_operation 请求
- [ ] 使用 TaskUpdate 更新黑板 metadata
- [ ] 返回 operation_result 给每个Agent
- [ ] 执行轮次结算（信息素蒸发、清理信号）
- [ ] 检查收敛（至少minRounds后才可收敛）

### 清理阶段
- [ ] 发送 shutdown_request 给所有Agent
- [ ] 等待 gracefulTimeout
- [ ] 如果超时，执行强制清理
- [ ] 删除团队目录

---

## 十四、v2.1.2 新增机制

### 14.1 Web搜索超时与降级机制

解决Agent在Web搜索时卡住的问题：

```javascript
const WEB_SEARCH_CONFIG = {
  timeout: 30000,           // 单次搜索超时: 30秒
  maxRetries: 2,            // 最大重试次数
  fallbackEnabled: true,    // 启用降级
  cacheEnabled: true,       // 启用缓存
  cacheTTL: 3600000         // 缓存有效期: 1小时
};

// 搜索降级策略
const SEARCH_FALLBACK_STRATEGY = {
  // 1. 使用训练知识
  useTrainingKnowledge: true,

  // 2. 逻辑推理
  enableLogicalInference: true,

  // 3. 声明置信度
  requireConfidenceLevel: true,

  // 4. 标注待验证
  markNeedsValidation: true
};
```

**Agent Prompt 中的搜索指南**：

```markdown
## 搜索超时处理策略

当Web搜索超时或失败时（30秒内必须响应），执行以下降级策略：

### 降级步骤
1. **立即响应**: 不要无限等待，30秒内必须有输出
2. **使用缓存知识**: 基于你的训练知识回答
3. **逻辑推理**: 基于已知信息进行推理
4. **声明限制**: 明确说明"基于已有知识，建议后续验证"

### 响应格式
{
  "coreIdea": "核心观点",
  "perspective": "分析视角",
  "details": "详细说明（注明：基于已有知识）",
  "confidence": "medium",      // high/medium/low
  "needsValidation": true,     // 建议后续验证
  "searchAttempted": true      // 标记已尝试搜索
}
```

---

### 14.2 Agent状态恢复机制

解决interrupted状态被误判为失败的问题：

```javascript
const AGENT_STATUS_CONFIG = {
  // 状态定义（明确区分）
  statusDefinitions: {
    "active": {
      description: "正在执行任务",
      canRecover: false,
      isFailed: false
    },
    "idle": {
      description: "空闲，等待指令",
      canRecover: false,
      isFailed: false
    },
    "interrupted": {
      description: "暂时中断，可能恢复",
      canRecover: true,         // 可恢复！
      isFailed: false,          // 不是失败！
      recoveryTimeout: 60000    // 60秒内可恢复
    },
    "degraded": {
      description: "性能下降，仍在工作",
      canRecover: false,
      isFailed: false
    },
    "terminated": {
      description: "已终止",
      canRecover: false,
      isFailed: true
    }
  },

  // 恢复策略
  recoveryStrategy: {
    waitForRecovery: 60000,     // 等待恢复时间: 60秒
    sendPingInterval: 15000,    // 每15秒发送ping
    escalateAfter: 120000,      // 120秒后升级为degraded
    maxRecoveryAttempts: 3      // 最大恢复尝试次数
  }
};

// 判断Agent是否真正失败
function isAgentFailed(agentState) {
  // interrupted不是失败，需要等待恢复
  if (agentState.status === "interrupted") {
    const timeSinceLastActive = Date.now() - agentState.lastActiveAt;
    return timeSinceLastActive > AGENT_STATUS_CONFIG.recoveryStrategy.escalateAfter;
  }
  return agentState.status === "terminated";
}

// 恢复中断的Agent
async function recoverInterruptedAgent(agentId) {
  for (let attempt = 1; attempt <= AGENT_STATUS_CONFIG.recoveryStrategy.maxRecoveryAttempts; attempt++) {
    // 发送恢复消息
    await SendMessage({
      type: "message",
      recipient: agentId,
      content: JSON.stringify({
        type: "recovery_request",
        attempt: attempt,
        message: "请继续你的探索任务"
      }),
      summary: `恢复Agent: ${agentId} (尝试 ${attempt})`
    });

    // 等待响应
    await sleep(AGENT_STATUS_CONFIG.recoveryStrategy.sendPingInterval);

    // 检查是否恢复
    const state = getAgentState(agentId);
    if (state.status === "active" || state.status === "idle") {
      return { recovered: true, agentId };
    }
  }

  // 恢复失败，标记为degraded
  await updateAgentStatus(agentId, "degraded");
  return { recovered: false, agentId, newStatus: "degraded" };
}
```

---

### 14.3 空闲Agent碰撞机制

让空闲的Agent与黑板上的发现进行"碰撞"，提升资源利用率：

```javascript
// 碰撞模式定义
const COLLISION_MODES = {
  // 模式1: 审查质疑
  REVIEW_CHALLENGE: {
    name: "review_challenge",
    description: "审查其他Agent的发现，提出质疑或补充",
    trigger: "agent_idle_with_findings",
    priority: 1
  },

  // 模式2: 观点辩论
  DEBATE: {
    name: "debate",
    description: "就某个有分歧的观点进行辩论",
    trigger: "divergent_views_detected",
    priority: 2
  },

  // 模式3: 协作深化
  COLLABORATE_DEEPEN: {
    name: "collaborate_deepen",
    description: "共同深化某个高价值发现",
    trigger: "high_pheromone_concentration",
    priority: 3
  },

  // 模式4: 早期综合
  EARLY_SYNTHESIS: {
    name: "early_synthesis",
    description: "尝试综合已有发现，生成草案",
    trigger: "round_complete_waiting",
    priority: 4
  }
};

// 碰撞规则
const COLLISION_RULES = {
  maxSameDirectionAgents: 2,     // 同一方向最多2个Agent
  encourageDivergence: true,     // 鼓励提出不同观点
  collisionWeights: {
    challenge: 1.5,              // 质疑权重最高
    support: 0.8,                // 支持权重较低
    extend: 1.2,                 // 延伸权重中等
    synthesize: 1.0              // 综合权重标准
  },
  triggerConditions: {
    minFindingsForCollision: 2,  // 至少2个发现
    minIdleAgents: 1,            // 至少1个空闲Agent
    maxCollisionPerRound: 3      // 每轮最多3次碰撞
  }
};
```

**碰撞消息协议**：

```javascript
// Orchestrator → 空闲Agent: 邀请碰撞
{
  "type": "collision_invitation",
  "mode": "review_challenge" | "debate" | "collaborate_deepen" | "early_synthesis",
  "targetFindings": [
    { "id": "finding-001", "coreIdea": "...", "byAgent": "SuYuan" }
  ],
  "blackboardSnapshot": {
    "pheromones": {...},
    "findings": [...]
  }
}

// Agent → Orchestrator: 碰撞响应
{
  "type": "collision_response",
  "mode": "review_challenge",
  "result": {
    "targetFindingId": "finding-001",
    "action": "challenge" | "support" | "extend" | "synthesize",
    "content": "审查/质疑/补充内容",
    "newInsights": ["新洞察1", "新洞察2"]
  }
}
```

**空闲Agent处理流程**：

```javascript
async function handleIdleAgents(blackboard, idleAgents) {
  if (idleAgents.length === 0) return;

  const findings = blackboard.metadata.findings;
  if (findings.length < COLLISION_RULES.triggerConditions.minFindingsForCollision) {
    return;  // 发现太少，无法碰撞
  }

  let collisionCount = 0;
  for (let agent of idleAgents) {
    if (collisionCount >= COLLISION_RULES.maxCollisionPerRound) break;

    // 选择碰撞模式
    const mode = selectCollisionMode(blackboard, agent);

    // 选择目标发现
    const targetFindings = selectTargetFindings(findings, agent, mode);

    // 发送碰撞邀请
    await SendMessage({
      type: "collision_invitation",
      recipient: agent.id,
      content: JSON.stringify({
        mode: mode,
        targetFindings: targetFindings,
        blackboardSnapshot: getBlackboardSnapshot(blackboard)
      }),
      summary: `碰撞邀请: ${mode}`
    });

    collisionCount++;
  }
}

function selectCollisionMode(blackboard, agent) {
  const pheromones = blackboard.metadata.pheromones;
  const findings = blackboard.metadata.findings;

  // 高浓度信息素 → 协作深化
  const maxConc = Math.max(...Object.values(pheromones).map(p => p.concentration || 0), 0);
  if (maxConc > 0.6) return "collaborate_deepen";

  // 有分歧观点 → 辩论
  const uniquePerspectives = new Set(findings.map(f => f.perspective));
  if (uniquePerspectives.size > 2) return "debate";

  // 默认 → 审查质疑
  return "review_challenge";
}
```

**Agent Prompt 中的碰撞指南**：

```markdown
## 空闲时碰撞行为

当你收到 `collision_invitation` 消息时，根据模式执行：

### 审查质疑模式 (review_challenge)
1. 阅读目标发现
2. 从你的视角评估：
   - 是否有逻辑漏洞？
   - 是否有遗漏视角？
   - 是否需要补充证据？
3. 选择 action: challenge / support / extend
4. 提交你的审查意见

### 协作深化模式 (collaborate_deepen)
1. 阅读高价值方向的已有发现
2. 贡献你独特的补充（避免重复）
3. 可添加新案例、新数据、新推理

### 辩论模式 (debate)
1. 选择与你观点不同的发现
2. 提出你的反驳或替代方案
3. 提供支持你观点的论据

### 响应格式
{
  "type": "collision_response",
  "mode": "review_challenge",
  "result": {
    "targetFindingId": "finding-001",
    "action": "challenge",
    "content": "你的审查意见",
    "newInsights": ["新洞察"]
  }
}
```

---

### 14.4 轮次恢复机制

当轮次因Agent中断而无法完成时，自动恢复：

```javascript
const ROUND_RECOVERY_CONFIG = {
  enableRecovery: true,
  recoveryTimeout: 90000,       // 90秒后尝试恢复
  maxRecoveryAttempts: 2,       // 最大恢复尝试次数
  degradeThreshold: 0.5         // 超过50% Agent失效才降级
};

async function recoverRound(blackboard, failedAgents) {
  const totalAgents = Object.keys(blackboard.metadata.agentStates).length;
  const failedRatio = failedAgents.length / totalAgents;

  // 如果超过阈值，降级处理
  if (failedRatio > ROUND_RECOVERY_CONFIG.degradeThreshold) {
    return {
      action: "degrade",
      message: `${failedAgents.length}/${totalAgents} Agent失效，降级处理`
    };
  }

  // 尝试恢复每个失败的Agent
  const recoveredAgents = [];
  for (let agentId of failedAgents) {
    const result = await recoverInterruptedAgent(agentId);
    if (result.recovered) {
      recoveredAgents.push(agentId);
    }
  }

  // 如果恢复足够多的Agent，继续当前轮次
  if (recoveredAgents.length > 0) {
    return {
      action: "continue",
      recoveredAgents: recoveredAgents,
      message: `恢复了 ${recoveredAgents.length} 个Agent`
    };
  }

  // 无法恢复，用已有发现继续
  return {
    action: "proceed_with_available",
    message: "使用可用Agent继续"
  };
}
```

---

### 14.5 更新后的执行流程

```
改进后的轮次流程:

┌─────────────────────────────────────────────────────────────┐
│ Round N                                                      │
├─────────────────────────────────────────────────────────────┤
│  1. 广播 round_start                                         │
│                                                              │
│  2. Agent执行探索                                            │
│     ├── 搜索超时 → 降级使用训练知识                          │
│     └── 完成探索 → 发送 round_complete                       │
│                                                              │
│  3. 【新增】空闲Agent碰撞                                    │
│     ├── 选择碰撞模式                                         │
│     ├── 发送 collision_invitation                            │
│     └── 处理 collision_response                               │
│                                                              │
│  4. 【新增】状态监控与恢复                                   │
│     ├── 检测 interrupted 状态                                │
│     ├── 发送 recovery_request                                │
│     └── 等待恢复或降级                                       │
│                                                              │
│  5. 处理黑板操作                                             │
│                                                              │
│  6. 轮次结算                                                 │
│                                                              │
│  7. 检查收敛（minRounds后才可收敛）                          │
└─────────────────────────────────────────────────────────────┘
```

---

### 14.6 更新后的检查清单 (v2.3)

每次运行Swarm时，Orchestrator必须完成以下步骤：

#### 初始化阶段
- [ ] `create_team` - TeamCreate 创建团队
- [ ] `create_blackboard` - TaskCreate 创建黑板（不设置owner）
- [ ] `init_agent_states` - 初始化所有Agent状态（包含内在阈值和随机探索概率）
- [ ] `spawn_agents` - 启动所有Explorer Agent
- [ ] **【v2.3】assertPhaseCompleted("INIT")**

#### 每轮执行
- [ ] `broadcast_round_start` - 发送轮次开始（含决策辅助）
- [ ] `wait_responses` - 等待Agent响应（使用同步屏障）
- [ ] **【v2.3】检测沉默Agent，发送force_wake**
- [ ] `process_operations` - **处理所有blackboard_operation**
- [ ] **【v2.3】验证操作队列：pending === processed + failed**
- [ ] `return_results` - **返回operation_result给每个Agent**
- [ ] `update_blackboard` - 使用TaskUpdate更新黑板
- [ ] `settle_round` - 信息素蒸发、清理信号
- [ ] **【v2.3】检查角色转换，发送role_transition_executed**
- [ ] `check_convergence` - 检查收敛（使用CONVERGENCE_CALCULATOR）
- [ ] **【v2.3】输出收敛计算具体数值**
- [ ] **【v2.3】markStepCompleted("ROUND", "round_N")**

#### 收敛/完成阶段
- [ ] **【v2.3】使用CONVERGENCE_CALCULATOR.evaluate()判断**
- [ ] **【v2.3】输出calculationDetails**
- [ ] `state_sync` - 执行状态同步协议
- [ ] `generate_report` - 生成最终报告
- [ ] **【v2.3】assertPhaseCompleted("CONVERGENCE")**

#### 清理阶段（三阶段关闭 - 必须全部执行）
- [ ] `pre_notify` - 阶段1：预通知 shutdown_imminent（5秒）
- [ ] `graceful_request` - 阶段2：优雅关闭 shutdown_request（15秒）
- [ ] **【v2.3】`force_terminate` - 阶段3：强制终止未响应Agent**
- [ ] **【v2.3】`mark_all_terminated` - 验证所有Agent状态为terminated**
- [ ] `update_blackboard_final` - 更新最终黑板状态
- [ ] **【v2.3】assertPhaseCompleted("SHUTDOWN")**
- [ ] TeamDelete 删除团队

---

### 14.7 v2.3 Orchestrator执行强制规则

```markdown
## 【v2.3 强制执行规则】

### 1. 黑板代理必须执行
- 收到 blackboard_operation → 立即处理 → 返回 operation_result
- 验证: pending === processed + failed

### 2. 信息素必须真实
- 禁止: 直接设置 pheromones[dir].concentration = X
- 必须: 通过 Agent 的 deposit_pheromone 操作

### 3. Shutdown第三阶段必须执行
- 即使所有Agent优雅关闭，也要执行 force_terminate
- 验证: 所有 agentStates.status === "terminated"

### 4. 收敛必须计算
- 禁止: 声称收敛但无计算过程
- 必须: 使用 CONVERGENCE_CALCULATOR.evaluate()
- 输出: calculationDetails 包含具体数值

### 5. 每步必须确认
- 每个关键步骤后: markStepCompleted(phase, step)
- 进入下一阶段前: assertPhaseCompleted(phase)
```

---

## 十五、v2.4 新增协议（强制执行 + 研究归档）

### 15.1 v2.4 核心改进

| 改进 | 说明 |
|------|------|
| **强制黑板代理** | Agent **必须**通过SendMessage发送操作，禁止在报告中声明 |
| **强制阈值计算** | 决策报告 **必须** 包含P(S,θ)计算过程和数值 |
| **强制冲突审查** | 每轮 **必须** 提交冲突审查报告，无冲突也要声明 |
| **合规性检查器** | `AGENT_COMPLIANCE_CHECKER` 验证Agent行为 |
| **违规处理器** | `VIOLATION_HANDLER` 累计违规分数，触发降级/终止 |
| **运行目录管理** | 每次运行创建独立目录，自动生成研究报告 |

### 15.2 运行目录结构（v2.4新增）

```
swarm-runs/
├── .template/                    # 模板目录
│   ├── README.md                 # 目录说明
│   └── report-template.md        # 报告模板
│
├── 2026-02-18-task-slug/         # 按日期+任务命名
│   ├── run-config.json           # 运行配置
│   ├── blackboard.json           # 黑板最终状态
│   ├── operation-log.json        # 操作日志
│   ├── agent-reports/            # Agent报告
│   │   ├── round-1/
│   │   │   ├── TanWei.md
│   │   │   └── ...
│   │   ├── round-2/
│   │   └── round-3/
│   ├── convergence-report.md     # 收敛报告
│   └── final-research-report.md  # 最终研究成果
│
└── ...
```

### 15.3 Agent合规性检查器（v2.4核心）

```javascript
const AGENT_COMPLIANCE_CHECKER = {
  checks: {
    // C1: 黑板操作必须通过SendMessage
    blackboardOperationViaMessage: {
      validate: (roundReport, operationLog) => {
        const reported = roundReport.confirmedOperations || [];
        const actual = operationLog.filter(op => op.from === roundReport.agentId);

        for (let op of reported) {
          const found = actual.find(a => a.operationId === op.operationId);
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

        const required = ["threshold", "candidates", "selectedDirection", "selectionReason"];
        for (let field of required) {
          if (!decision[field]) {
            return { valid: false, violation: `decision_report_missing_${field}` };
          }
        }

        // 验证响应概率计算
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
        }
        return { valid: true };
      }
    }
  },

  runChecks: (roundReport, context) => {
    const results = [];
    for (let [checkName, check] of Object.entries(AGENT_COMPLIANCE_CHECKER.checks)) {
      const result = check.validate(roundReport, context);
      results.push({ check: checkName, ...result });
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

### 15.4 违规处理器（v2.4核心）

```javascript
const VIOLATION_HANDLER = {
  levels: {
    WARNING: { score: 1, action: "record_only" },
    MINOR: { score: 3, action: "reduce_pheromone_weight" },
    MAJOR: { score: 5, action: "degrade_agent" },
    CRITICAL: { score: 10, action: "force_terminate" }
  },

  violationSeverity: {
    "reported_operation_not_found": "MAJOR",
    "decision_report_missing": "MAJOR",
    "decision_report_missing_threshold": "MINOR",
    "response_prob_calculation_error": "MINOR",
    "conflict_review_missing": "MINOR",
    "random_explore_not_executed": "MAJOR"
  },

  handle: async (blackboard, agentId, violations) => {
    const state = blackboard.metadata.agentStates[agentId];
    state.violations = state.violations || [];
    state.violationScore = state.violationScore || 0;

    for (let violation of violations) {
      const severity = VIOLATION_HANDLER.violationSeverity[violation.violation] || "WARNING";
      const level = VIOLATION_HANDLER.levels[severity];

      state.violations.push({ ...violation, severity, timestamp: Date.now() });
      state.violationScore += level.score;

      console.log(`[COMPLIANCE] ${agentId} ${severity}: ${violation.violation}`);

      if (level.action === "degrade_agent" && state.status === "active") {
        state.status = "degraded";
      }

      if (level.action === "force_terminate" || state.violationScore >= 15) {
        state.status = "terminated";
        state.terminationReason = "compliance_violation";
      }
    }
    return state;
  }
};
```

### 15.5 自动强制终止机制（v2.4核心）

```javascript
const AUTO_FORCE_TERMINATE = {
  triggers: {
    shutdownRequestTimeout: 30000,
    maxShutdownRetries: 3,
    agentSilentTime: 120000
  },

  execute: async (teamName, blackboardId, agentId) => {
    console.log(`[FORCE TERMINATE] 自动终止 ${agentId}`);

    // 1. 更新黑板状态
    const blackboard = await getBlackboard(blackboardId);
    if (blackboard.metadata.agentStates[agentId]) {
      blackboard.metadata.agentStates[agentId].status = "terminated";
      blackboard.metadata.agentStates[agentId].terminatedAt = Date.now();
      blackboard.metadata.agentStates[agentId].terminationReason = "auto_forced";
    }

    // 2. 更新团队配置
    const configPath = `~/.claude/teams/${teamName}/config.json`;
    const config = JSON.parse(await readFile(configPath));
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
    console.log(`[FORCE TERMINATE] ✓ ${agentId} 已自动终止`);

    return { success: true, agentId, method: "auto_forced" };
  },

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
```

### 15.6 运行目录管理器（v2.4核心）

```javascript
const RUN_DIRECTORY_MANAGER = {
  createRunDirectory: async (taskName) => {
    const date = new Date().toISOString().slice(0, 10);
    const taskSlug = taskName.toLowerCase().replace(/[^a-z0-9\u4e00-\u9fff]+/g, '-').slice(0, 30);
    const runId = `${date}-${taskSlug}`;
    const runPath = `swarm-runs/${runId}`;

    await Bash(`mkdir -p "${runPath}/agent-reports"`);
    return { runId, runPath };
  },

  saveRunConfig: async (runPath, config) => {
    await writeFile(`${runPath}/run-config.json`, JSON.stringify(config, null, 2));
  },

  saveBlackboard: async (runPath, blackboard) => {
    await writeFile(`${runPath}/blackboard.json`, JSON.stringify(blackboard, null, 2));
  },

  generateFinalReport: async (runPath, blackboard, convergenceResult, task) => {
    // 生成人类可读的研究报告
    const report = generateMarkdownReport(blackboard, convergenceResult, task);
    await writeFile(`${runPath}/final-research-report.md`, report);
    return `${runPath}/final-research-report.md`;
  }
};
```

---

### 15.7 更新后的检查清单 (v2.4)

#### 初始化阶段
- [ ] `create_run_directory` - 创建运行目录
- [ ] `create_team` - TeamCreate 创建团队
- [ ] `create_blackboard` - TaskCreate 创建黑板（含runPath）
- [ ] `init_agent_states` - 初始化所有Agent状态（含违规分数）
- [ ] `spawn_agents` - 启动所有Explorer Agent
- [ ] **【v2.4】saveRunConfig** - 保存运行配置

#### 每轮执行
- [ ] `broadcast_round_start` - 发送轮次开始
- [ ] `wait_responses` - 等待Agent响应
- [ ] **【v2.4】runComplianceChecks** - 运行合规性检查
- [ ] **【v2.4】handleViolations** - 处理违规（如有）
- [ ] `process_operations` - 处理blackboard_operation
- [ ] `return_results` - 返回operation_result
- [ ] `settle_round` - 轮次结算
- [ ] **【v2.4】saveAgentReports** - 保存Agent报告

#### 收敛/完成阶段
- [ ] `check_convergence` - 使用CONVERGENCE_CALCULATOR
- [ ] `generate_convergence_report` - 生成收敛报告
- [ ] **【v2.4】generateFinalReport** - 生成最终研究报告
- [ ] `state_sync` - 状态同步

#### 清理阶段
- [ ] `pre_notify` - 预通知
- [ ] `graceful_request` - 优雅关闭
- [ ] **【v2.4】auto_force_terminate** - 自动强制终止
- [ ] **【v2.4】saveBlackboard** - 保存黑板最终状态
- [ ] TeamDelete 删除团队

---

### 15.8 v2.4 Orchestrator执行强制规则

```markdown
## 【v2.4 强制执行规则】

### 1. Agent必须通过SendMessage操作黑板
- 禁止: 在round_complete报告中声明操作
- 必须: 通过SendMessage发送blackboard_operation
- 验证: AGENT_COMPLIANCE_CHECKER.blackboardOperationViaMessage

### 2. 决策报告必须包含阈值计算
- 必须: decisionReport.threshold, candidates, selectedDirection
- 必须: 候选方向的responseProb = S²/(S²+θ²)
- 验证: AGENT_COMPLIANCE_CHECKER.decisionReportRequired

### 3. 冲突审查必须每轮执行
- 必须: 每轮提交conflictReview报告
- 即使无冲突也必须声明: { conflictsFound: 0 }
- 验证: AGENT_COMPLIANCE_CHECKER.conflictReviewRequired

### 4. 违规累计触发惩罚
- MAJOR违规: +5分，触发降级
- 累计≥15分: 强制终止
- 验证: VIOLATION_HANDLER

### 5. 研究成果必须归档
- 必须: 创建运行目录
- 必须: 生成final-research-report.md
- 验证: 检查文件存在
```

---

## 十六、版本历史

| 版本 | 日期 | 主要改进 |
|------|------|----------|
| v1.0 | 2024 | 基础并行探索 + 视角菜单 |
| v2.0 | 2026-02 | 数字信息素、交叉抑制、量化共识、动态角色 |
| v2.1 | 2026-02-17 | 黑板代理、状态托管、多轮迭代、超时降级 |
| v2.1.1 | 2026-02-18 | 强制清理、同步屏障、最少轮数限制 |
| v2.1.2 | 2026-02-18 | Web搜索降级、Agent恢复、空闲碰撞 |
| v2.2 | 2026-02-18 | 理论符合性修复：状态同步、三阶段关闭、停止信号强化 |
| v2.3 | 2026-02-18 | 执行层修复：操作队列、信息素真实化、收敛计算器 |
| **v2.4** | 2026-02-18 | **强制协议执行** + **合规检查器** + **运行目录管理** + **研究报告生成** |

---

*Swarm v2.4 - 基于群体智能理论的自组织协作系统 | 强制执行版本*
