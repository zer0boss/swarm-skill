# 动态孵化 Agent 设计方案

## 一、核心概念

### 1.1 孵化触发条件

Agent 在探索过程中检测到"知识漏洞"时，可请求孵化专业 Agent：

| 漏洞类型 | 示例 | 孵化目标 |
|---------|------|---------|
| **专业领域缺失** | 发现涉及法律问题，但无法律专家 | 领域专家 Agent |
| **技能缺失** | 需要代码执行，但当前只有分析能力 | 工具型 Agent |
| **信息源缺失** | 需要访问特定 API/数据库 | 数据采集 Agent |
| **视角缺失** | 发现缺少批判性视角 | 辩论型 Agent |

### 1.2 孵化协议

```javascript
// Agent → Orchestrator：请求孵化
{
  type: "blackboard_operation",
  operation: "request_spawn",
  params: {
    specialization: "legal_expert",     // 专业化类型
    reason: "发现法律合规问题需要专业分析",  // 原因
    context: "GDPR第5条关于数据最小化原则...", // 上下文
    urgency: "medium",                  // 紧急程度
    suggestedCapabilities: ["legal_analysis", "compliance_check"]
  }
}

// Orchestrator → Agent：孵化结果
{
  type: "spawn_result",
  success: true,
  spawnedAgentId: "specialist-legal-001",
  capabilities: ["legal_analysis", "compliance_check"],
  lifespan: "task_completion"  // 任务完成后自动终止
}
```

## 二、实现细节

### 2.1 黑板操作扩展

```javascript
const BLACKBOARD_OPERATIONS = {
  // ... 现有操作 ...

  // 新增：请求孵化
  REQUEST_SPAWN: {
    name: "request_spawn",
    params: {
      specialization: "string",      // 专业化类型
      reason: "string",              // 原因
      context: "string",             // 上下文
      urgency: "low" | "medium" | "high",
      suggestedCapabilities: ["string"]
    },
    handler: (blackboard, params, agentId) => {
      // 1. 检查孵化限制
      const spawnConfig = blackboard.metadata.config.spawnConfig;
      const currentCount = Object.keys(blackboard.metadata.agentStates).length;

      if (currentCount >= spawnConfig.maxTotalAgents) {
        return { success: false, reason: "max_agents_reached" };
      }

      // 2. 检查是否已存在类似专业 Agent
      const existingSpecialist = findSimilarSpecialist(
        blackboard,
        params.specialization
      );

      if (existingSpecialist) {
        return {
          success: true,
          spawnedAgentId: existingSpecialist,
          reused: true,
          message: "已存在类似专业Agent，复用中"
        };
      }

      // 3. 记录孵化请求
      blackboard.metadata.spawnRequests = blackboard.metadata.spawnRequests || [];
      blackboard.metadata.spawnRequests.push({
        requestId: `spawn-${Date.now()}`,
        from: agentId,
        ...params,
        timestamp: Date.now(),
        status: "pending"
      });

      return {
        success: true,
        status: "pending",
        message: "孵化请求已提交，等待Orchestrator处理"
      };
    }
  }
};
```

### 2.2 Orchestrator 孵化处理

```javascript
const SPAWN_CONFIG = {
  maxTotalAgents: 12,           // 最大 Agent 总数
  maxSpawnPerRound: 2,          // 每轮最多孵化数
  specializations: {
    "legal_expert": {
      description: "法律合规分析专家",
      capabilities: ["legal_analysis", "compliance_check", "risk_assessment"],
      promptTemplate: "LEGAL_EXPERT_PROMPT"
    },
    "data_analyst": {
      description: "数据分析专家",
      capabilities: ["statistical_analysis", "data_visualization", "trend_detection"],
      promptTemplate: "DATA_ANALYST_PROMPT"
    },
    "technical_auditor": {
      description: "技术审计专家",
      capabilities: ["code_review", "security_audit", "performance_analysis"],
      promptTemplate: "TECHNICAL_AUDITOR_PROMPT"
    },
    "domain_researcher": {
      description: "领域研究员",
      capabilities: ["literature_review", "expert_interview", "trend_forecast"],
      promptTemplate: "DOMAIN_RESEARCHER_PROMPT"
    }
  },
  lifespanPolicy: {
    default: "task_completion",    // 任务完成后终止
    maxIdleRounds: 2,              // 空闲2轮后终止
    criticalOnly: false            // 是否仅保留关键专家
  }
};

async function processSpawnRequests(blackboard, teamName) {
  const requests = blackboard.metadata.spawnRequests?.filter(r => r.status === "pending") || [];
  const config = blackboard.metadata.config.spawnConfig;
  let spawned = 0;

  for (let request of requests) {
    // 检查每轮限制
    if (spawned >= config.maxSpawnPerRound) break;

    // 检查总数限制
    const currentCount = Object.keys(blackboard.metadata.agentStates).length;
    if (currentCount >= config.maxTotalAgents) {
      request.status = "rejected";
      request.rejectReason = "max_agents_reached";
      continue;
    }

    // 获取专业化配置
    const specConfig = config.specializations[request.specialization];
    if (!specConfig) {
      request.status = "rejected";
      request.rejectReason = "unknown_specialization";
      continue;
    }

    // 生成 Agent ID
    const agentId = `specialist-${request.specialization}-${Date.now()}`;

    // 初始化 Agent 状态
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

    // 启动 Agent
    await Task({
      subagent_type: "Explore",
      name: agentId,
      team_name: teamName,
      prompt: generateSpecialistPrompt(specConfig, request, blackboard)
    });

    // 更新请求状态
    request.status = "completed";
    request.spawnedAgentId = agentId;
    spawned++;

    console.log(`[SPAWN] 孵化成功: ${agentId} (${request.specialization})`);
  }

  return { spawned, requests: requests.filter(r => r.status === "completed") };
}

function generateSpecialistPrompt(specConfig, request, blackboard) {
  return `
你是 Swarm 系统中被动态孵化的**${specConfig.description}**。

## 孵化背景
- **孵化者**: ${request.from}
- **孵化原因**: ${request.reason}
- **相关上下文**: ${request.context}

## 你的专业能力
${specConfig.capabilities.map(c => `- ${c}`).join('\n')}

## 你的职责
1. 专注于你的专业领域，解决孵化原因中描述的问题
2. 完成任务后，通过 update_finding 提交你的专业分析
3. 你是临时 Agent，任务完成后将自动终止

## 当前黑板状态
- 主要发现: ${JSON.stringify(blackboard.metadata.findings.slice(-3))}
- 高信息素方向: ${JSON.stringify(Object.entries(blackboard.metadata.pheromones).slice(0, 3))}

## 原始任务
${blackboard.metadata.taskDescription}
`;
}
```

### 2.3 生命周期管理

```javascript
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

async function terminateIdleSpecialists(blackboard, teamName) {
  const toTerminate = Object.entries(blackboard.metadata.agentStates)
    .filter(([_, state]) => state.status === "pending_termination");

  for (let [agentId, state] of toTerminate) {
    // 发送终止通知
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

    // 更新状态
    state.status = "terminated";
    state.terminatedAt = Date.now();
  }
}
```

## 三、集成到现有流程

### 3.1 修改执行流程

```javascript
async function runSwarm() {
  while (currentRound < config.maxRounds) {
    currentRound++;

    // 1. 广播轮次开始
    await broadcastRoundStart();

    // 2. 等待Agent响应
    const responses = await waitForResponses();

    // 3. 处理黑板操作（包含 request_spawn）
    await processOperations(responses);

    // 【新增】4. 处理孵化请求
    const spawnResults = await processSpawnRequests(blackboard, teamName);
    if (spawnResults.spawned > 0) {
      // 通知所有 Agent 有新成员加入
      await broadcastNewMembers(spawnResults.requests);
    }

    // 5. 轮次结算
    await settleRound();

    // 【新增】6. 管理专业 Agent 生命周期
    await manageSpecialistLifespan(blackboard);
    await terminateIdleSpecialists(blackboard, teamName);

    // 7. 检查收敛
    if (checkConvergence().converged) {
      return generateFinalReport();
    }
  }
}
```

### 3.2 更新配置结构

```javascript
const DEFAULT_CONFIG = {
  // ... 现有配置 ...

  // 新增：动态孵化配置
  spawnConfig: SPAWN_CONFIG
};
```

## 四、Agent Prompt 扩展

在 Explorer Prompt 中添加：

```markdown
## 动态孵化能力

当你在探索过程中发现需要专业知识才能解决的问题时，可以请求孵化专业 Agent：

### 触发条件
- 发现超出你专业能力的问题
- 需要特定领域知识（如法律、技术审计、数据分析）
- 需要特定工具或数据源

### 请求格式
SendMessage({
  type: "blackboard_operation",
  operation: "request_spawn",
  params: {
    specialization: "legal_expert" | "data_analyst" | "technical_auditor" | "domain_researcher",
    reason: "为什么需要这个专业Agent",
    context: "相关背景信息",
    urgency: "low" | "medium" | "high"
  }
})

### 注意事项
- 不要滥用孵化请求，每个方向最多请求1个专业Agent
- 在请求前确认黑板中不存在类似的专业Agent
- 专业Agent是临时的，任务完成后会自动终止
```

## 五、预期效果

### 5.1 正面效果
- **适应性增强**：系统可以应对意外的专业问题
- **效率提升**：专业Agent专注于特定问题
- **涌现增强**：更多样化的视角和能力

### 5.2 风险控制
- **资源限制**：最大Agent数量限制（默认12个）
- **生命周期**：空闲超时自动终止（默认2轮）
- **去重机制**：复用已存在的类似专业Agent

## 六、示例场景

```
Round 3:
TanWei 在探索中发现：
  "这个方案涉及GDPR合规问题，需要法律专家评估"

TanWei 发送 request_spawn:
  specialization: "legal_expert"
  reason: "GDPR合规评估"
  context: "用户数据处理方案..."

Orchestrator:
  1. 检查：无现有法律专家
  2. 孵化：specialist-legal-expert-xxx
  3. 启动：使用 LEGAL_EXPERT_PROMPT

Round 4:
specialist-legal-expert-xxx:
  1. 分析GDPR合规问题
  2. 提交专业发现
  3. 建议：需要修改数据处理流程

Round 5-6:
specialist-legal-expert-xxx:
  - 贡献数 > 0，继续活跃

Round 7:
specialist-legal-expert-xxx:
  - 无新贡献，空闲计数+1

Round 8:
specialist-legal-expert-xxx:
  - 空闲达到2轮，标记终止
  - 发送感谢消息后终止
```
