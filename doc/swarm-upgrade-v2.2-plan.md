# Swarm v2.2 升级方案

> 基于v2.1.2实际运行复盘，修复不符合理论范式的问题

**版本**: v2.2.0
**日期**: 2026-02-18
**基于**: Swarm v2.1.2 复盘分析

---

## 一、问题清单与优先级

### 1.1 高优先级问题（影响任务执行）

| # | 问题 | 表现 | 理论要求 | 影响 |
|---|------|------|----------|------|
| P1 | Agent状态同步失效 | 任务状态pending但报告已生成 | 任务完成=状态更新 | 用户困惑 |
| P2 | Shutdown流程不完整 | Agent idle循环、TeamDelete失败 | 优雅关闭所有Agent | 资源泄漏 |
| P3 | 停止信号无效 | 信号发送但不影响Agent决策 | 抑制目标方向30% | 过早收敛 |

### 1.2 中优先级问题（影响结果质量）

| # | 问题 | 表现 | 理论要求 | 影响 |
|---|------|------|----------|------|
| P4 | 随机探索未实现 | 所有Agent跟随信息素 | 15%随机探索 | 多样性不足 |
| P5 | 角色转换未执行 | 全程Explorer无演化 | 阈值驱动动态转换 | 分工缺失 |
| P6 | 多样性保护缺失 | 2轮即100%共识 | 40%最低多样性保证 | 探索不足 |

### 1.3 低优先级问题（优化项）

| # | 问题 | 表现 | 理论要求 | 影响 |
|---|------|------|----------|------|
| P7 | 信息素蒸发弱 | 浓度快速饱和 | 保持梯度 | 区分度下降 |
| P8 | 任务分解缺失 | 预设单一任务 | 涌现式分解 | 灵活性不足 |

---

## 二、升级方案

### 2.1 P1: Agent状态同步修复

**问题**: 任务状态与实际执行状态不同步

**解决方案**: 引入状态同步协议

```javascript
// 新增: 状态同步协议
const STATE_SYNC_PROTOCOL = {
  // 任务完成时必须执行的步骤
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

**实施位置**: Orchestrator收敛判断后立即执行

---

### 2.2 P2: Shutdown流程强化

**问题**: Agent未正确关闭，导致idle循环

**解决方案**: 三阶段关闭协议

```javascript
// 新增: 三阶段关闭协议
const SHUTDOWN_PROTOCOL = {
  phases: {
    // 阶段1: 预通知（5秒）
    preNotify: {
      timeout: 5000,
      action: "broadcast_shutdown_imminent",
      message: "协作即将结束，请保存工作"
    },

    // 阶段2: 正式关闭请求（15秒）
    gracefulRequest: {
      timeout: 15000,
      action: "send_shutdown_request",
      expectResponse: true
    },

    // 阶段3: 强制终止（10秒后）
    forceTerminate: {
      timeout: 10000,
      action: "force_cleanup",
      fallback: true
    }
  },

  // 执行关闭
  execute: async (agents, blackboard) => {
    const results = { graceful: [], forced: [], failed: [] };

    // 阶段1: 预通知
    await broadcast({
      type: "shutdown_imminent",
      reason: "协作完成",
      prepareTime: 5000
    });
    await sleep(5000);

    // 阶段2: 正式请求
    const shutdownPromises = agents.map(agent =>
      sendShutdownRequest(agent.id, 15000)
        .then(result => {
          if (result.acknowledged) {
            results.graceful.push(agent.id);
          } else {
            results.failed.push(agent.id);
          }
        })
    );
    await Promise.allSettled(shutdownPromises);

    // 阶段3: 强制清理
    const remainingAgents = agents.filter(a =>
      !results.graceful.includes(a.id) &&
      !results.forced.includes(a.id)
    );

    if (remainingAgents.length > 0) {
      console.warn(`强制终止 ${remainingAgents.length} 个Agent`);
      results.forced = remainingAgents.map(a => a.id);

      // 清理Agent状态
      for (let agentId of results.forced) {
        if (blackboard.metadata.agentStates[agentId]) {
          blackboard.metadata.agentStates[agentId].status = "terminated";
          blackboard.metadata.agentStates[agentId].terminatedAt = Date.now();
          blackboard.metadata.agentStates[agentId].terminationReason = "forced";
        }
      }
    }

    // 等待系统清理
    await sleep(2000);

    return {
      success: true,
      graceful: results.graceful.length,
      forced: results.forced.length,
      failed: results.failed.length
    };
  }
};
```

**实施位置**: Orchestrator收敛/超时后执行

---

### 2.3 P3: 停止信号强化

**问题**: 信号发送但不影响Agent决策

**解决方案**: 停止信号必须影响信息素有效浓度

```javascript
// 修改: 黑板操作 - 停止信号处理
const ENHANCED_STOP_SIGNAL = {
  // 发送停止信号时，立即降低目标方向的信息素
  handler: (blackboard, params, agentId) => {
    const signal = {
      id: `signal-${Date.now()}`,
      from: agentId,
      target: params.targetDirection,
      reason: params.reason,
      evidence: params.evidence,
      strength: 0.3,  // 固定30%抑制
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

    // 更新Agent统计
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
      s => s.target === direction && s.active && Date.now() - s.timestamp < 300000
    );

    if (signals.length === 0) return { inhibited: false };

    // 计算有效浓度
    const totalStrength = Math.min(
      signals.reduce((sum, s) => sum + s.strength, 0),
      0.5  // 最大50%
    );

    const rawConc = blackboard.metadata.pheromones[direction]?.concentration || 0;
    const effectiveConc = rawConc * (1 - totalStrength);

    // 阈值响应判断
    const state = blackboard.metadata.agentStates[agentId];
    const responseProb = responseProbability(effectiveConc, state.internalThreshold);

    return {
      inhibited: Math.random() > responseProb,
      totalStrength,
      rawConc,
      effectiveConc,
      responseProb,
      shouldSwitch: effectiveConc < state.internalThreshold
    };
  }
};
```

**Agent Prompt 更新**: 明确告知抑制效果

```markdown
## 停止信号处理（重要）

当你检查发现你正在探索的方向收到停止信号时：

1. **查看有效浓度**: effectiveConc = 原始浓度 × (1 - 抑制强度)
2. **与阈值比较**: 如果 effectiveConc < 你的 internalThreshold
3. **必须切换**: 你应该切换到其他方向

这是强制性的多样性保护机制，防止过早趋同！
```

---

### 2.4 P4: 随机探索强制实现

**问题**: 理论要求15%随机探索，实际未执行

**解决方案**: 在round_start中明确指令随机探索

```javascript
// 修改: broadcastRoundStart
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
        blackboardSnapshot: {
          pheromones: blackboard.metadata.pheromones,
          stopSignals: blackboard.metadata.stopSignals,
          findings: blackboard.metadata.findings,
          claims: blackboard.metadata.claims
        },
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

**Agent Prompt 更新**:

```markdown
## 轮次指令（必须遵守）

收到 round_start 消息后，检查 instructions 字段：

### 情况1: forceRandomExplore = true
你**必须**忽略信息素，随机选择一个探索方向。
这是为了保证系统多样性，防止过早收敛。

### 情况2: mustSwitchDirection = true
你当前的方向被停止信号抑制，**必须**切换到其他方向。
参考：你的阈值 = {internalThreshold}，有效浓度 = {effectiveConc}

### 情况3: 正常模式
按照信息素浓度选择方向：
- 70% 概率跟随高浓度方向
- 30% 概率选择低覆盖方向
```

---

### 2.5 P5: 角色转换强制执行

**问题**: 角色转换条件存在但未执行

**解决方案**: 在轮次结算时强制检查并通知

```javascript
// 修改: settleRound - 角色转换强制执行
async function settleRound(blackboard) {
  // ... 其他结算逻辑 ...

  // 【强制】检查并执行角色转换
  const transitions = checkRoleTransitions(blackboard);
  const executedTransitions = [];

  for (let transition of transitions) {
    const state = blackboard.metadata.agentStates[transition.agentId];

    // 验证转换条件
    if (transition.toRole === 'DEEP_ANALYST') {
      const maxConc = Math.max(
        ...Object.values(blackboard.metadata.pheromones).map(p => p.concentration),
        0
      );
      if (maxConc < 0.7 || state.stats.pheromoneDeposits < 3) continue;
    }

    // 执行转换
    const oldRole = state.role;
    state.role = transition.toRole;
    state.roleHistory = state.roleHistory || [];
    state.roleHistory.push({
      from: oldRole,
      to: transition.toRole,
      reason: transition.reason,
      round: currentRound,
      timestamp: Date.now()
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
      summary: `角色转换执行: ${oldRole} → ${transition.toRole}`
    });

    executedTransitions.push(transition);
  }

  // 记录转换事件
  blackboard.metadata.roleTransitionLog = blackboard.metadata.roleTransitionLog || [];
  blackboard.metadata.roleTransitionLog.push({
    round: currentRound,
    transitions: executedTransitions,
    timestamp: Date.now()
  });

  return { transitionsExecuted: executedTransitions.length };
}

// 角色能力定义
const ROLE_CAPABILITIES = {
  DEEP_ANALYST: {
    description: "深入分析高价值方向",
    canDo: ["deep_dive", "strengthen_pheromone", "call_for_validation"],
    focusOn: "最高信息素浓度的方向"
  },
  DEBATER: {
    description: "挑战主流观点",
    canDo: ["send_stop_signal", "propose_alternative", "cross_examine"],
    focusOn: "寻找主流观点的漏洞"
  },
  SYNTHESIZER: {
    description: "整合多个观点",
    canDo: ["merge_findings", "identify_patterns", "generate_summary"],
    focusOn: "综合所有发现"
  }
};

// 角色指令
function getRoleInstructions(role) {
  const instructions = {
    DEEP_ANALYST: `
你现在被提升为 DeepAnalyst。你的任务是：
1. 深入分析最高信息素浓度的方向
2. 寻找该方向的深层逻辑和数据支持
3. 可以强化该方向的信息素（每次额外+0.05）
4. 完成后建议是否需要验证
    `,
    DEBATER: `
你现在被激活为 Debater。你的任务是：
1. 审查主流观点，寻找逻辑漏洞
2. 发送停止信号挑战你认为不充分的方向
3. 提出替代方案或反例
4. 保持系统的批判性思维
    `,
    SYNTHESIZER: `
你现在被委派为 Synthesizer。你的任务是：
1. 整合所有Agent的发现
2. 识别共性和分歧
3. 生成阶段性综合报告
4. 为最终收敛做准备
    `
  };
  return instructions[role] || "";
}
```

---

### 2.6 P6: 多样性保护强制执行

**问题**: 2轮即100%共识，探索不足

**解决方案**: 引入多样性保护阈值

```javascript
// 新增: 多样性保护机制
const DIVERSITY_PROTECTION = {
  minRoundsBeforeConvergence: 3,  // 最少3轮才能收敛
  minDiversityThreshold: 0.4,     // 最低多样性40%
  maxConsensusRate: 0.9,          // 单轮最大共识率90%

  // 计算多样性
  calculateDiversity: (blackboard) => {
    const findings = blackboard.metadata.findings;
    const pheromones = blackboard.metadata.pheromones;

    // 1. 视角多样性
    const perspectives = new Set(findings.map(f => f.perspective).filter(Boolean));
    const perspectiveDiversity = perspectives.size / 8;

    // 2. 观点正交性
    const ideas = findings.map(f => f.coreIdea).filter(Boolean);
    const uniqueIdeas = new Set(ideas);
    const orthogonality = uniqueIdeas.size / Math.max(ideas.length, 1);

    // 3. 信息素分布熵
    const concentrations = Object.values(pheromones).map(p => p.concentration);
    const totalConc = concentrations.reduce((a, b) => a + b, 0);
    const entropy = totalConc > 0
      ? -concentrations.reduce((sum, c) => {
          const p = c / totalConc;
          return sum + (p > 0 ? p * Math.log2(p) : 0);
        }, 0) / Math.log2(Math.max(concentrations.length, 2))
      : 0;

    return {
      perspectiveDiversity,
      orthogonality,
      entropy,
      overall: (perspectiveDiversity + orthogonality + entropy) / 3
    };
  },

  // 检查是否允许收敛
  canConverge: (blackboard, currentRound) => {
    // 1. 最少轮数检查
    if (currentRound < DIVERSITY_PROTECTION.minRoundsBeforeConvergence) {
      return {
        allowed: false,
        reason: `min_rounds_not_reached: ${currentRound}/${DIVERSITY_PROTECTION.minRoundsBeforeConvergence}`
      };
    }

    // 2. 多样性检查
    const diversity = DIVERSITY_PROTECTION.calculateDiversity(blackboard);
    if (diversity.overall < DIVERSITY_PROTECTION.minDiversityThreshold) {
      return {
        allowed: false,
        reason: `diversity_too_low: ${diversity.overall.toFixed(2)}/${DIVERSITY_PROTECTION.minDiversityThreshold}`,
        diversity: diversity
      };
    }

    // 3. 共识率检查
    const consensusRate = calculateConsensusRate(blackboard);
    if (consensusRate > DIVERSITY_PROTECTION.maxConsensusRate && currentRound < 5) {
      return {
        allowed: false,
        reason: `consensus_too_fast: ${consensusRate.toFixed(2)}>${DIVERSITY_PROTECTION.maxConsensusRate}`,
        suggestion: "激活Debater角色挑战主流观点"
      };
    }

    return {
      allowed: true,
      diversity: diversity,
      consensusRate: consensusRate
    };
  },

  // 保护动作
  protect: async (blackboard, reason) => {
    const actions = [];

    // 1. 强制随机探索
    for (let agentId in blackboard.metadata.agentStates) {
      const state = blackboard.metadata.agentStates[agentId];
      state.forceRandomExplore = true;
    }
    actions.push("all_agents_force_random");

    // 2. 激活Debater
    const explorers = Object.entries(blackboard.metadata.agentStates)
      .filter(([_, s]) => s.role === "EXPLORER" && s.status === "active");

    if (explorers.length > 0) {
      const [debaterId, debaterState] = explorers[0];
      debaterState.role = "DEBATER";
      debaterState.roleHistory = debaterState.roleHistory || [];
      debaterState.roleHistory.push({
        from: "EXPLORER",
        to: "DEBATER",
        reason: "diversity_protection",
        timestamp: Date.now()
      });
      actions.push(`activated_debater:${debaterId}`);
    }

    // 3. 降低主导信息素
    const pheromones = blackboard.metadata.pheromones;
    const maxDir = Object.entries(pheromones)
      .sort((a, b) => b[1].concentration - a[1].concentration)[0];
    if (maxDir && maxDir[1].concentration > 0.7) {
      maxDir[1].concentration *= 0.7;  // 降低30%
      actions.push(`suppressed_dominant:${maxDir[0]}`);
    }

    return { actions, reason };
  }
};
```

---

## 三、配置参数更新

```javascript
// v2.2 默认配置
const SWARM_CONFIG_V22 = {
  // 信息素
  evaporationRate: 0.08,      // 提高到8%（原5%）
  depositAmount: 0.1,
  maxConcentration: 1.0,
  minThreshold: 0.1,

  // 共识
  betaStability: 2,
  quorumThreshold: 0.67,
  minDiversity: 0.4,

  // 轮次
  minRounds: 3,               // 提高到3（原2）
  maxRounds: 10,
  roundTimeout: 120000,
  responseTimeout: 60000,

  // 关闭
  shutdownTimeout: 30000,
  preNotifyTimeout: 5000,
  gracefulTimeout: 15000,
  forceCleanupTimeout: 10000,

  // 多样性保护
  minRoundsBeforeConvergence: 3,
  maxConsensusRate: 0.9,

  // 随机探索
  randomExploreProbMin: 0.1,
  randomExploreProbMax: 0.2
};
```

---

## 四、实施计划

### Phase 1: 核心修复（立即实施）
- [x] P1: 状态同步协议
- [x] P2: 三阶段关闭协议
- [x] P3: 停止信号强化

### Phase 2: 机制增强（同步实施）
- [x] P4: 随机探索强制
- [x] P5: 角色转换执行
- [x] P6: 多样性保护

### Phase 3: Skill文件更新
- [ ] 更新 skill.md 到 v2.2
- [ ] 更新 Orchestrator Prompt
- [ ] 更新 Explorer Prompt
- [ ] 更新 Validator Prompt

---

## 五、验证清单

升级后必须验证：

1. **状态同步**: 任务完成后状态变为completed
2. **关闭流程**: 所有Agent正确终止，无idle循环
3. **停止信号**: 被抑制方向的信息素确实降低
4. **随机探索**: 有Agent执行随机探索
5. **角色转换**: 有Agent角色发生变化
6. **多样性**: 收敛时多样性≥40%

---

*Swarm v2.2 升级方案 | 2026-02-18*
