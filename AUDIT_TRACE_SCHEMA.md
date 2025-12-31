# 审计字段合同 | Audit Trace Schema

> **目的**：把决策链固化成"可验证合同"，让每次路由/回退都可追责。  
> **适用场景**：Agent 执行、自动化工作流、高风险工具调用、边缘推理路由。

---

## 一、为什么需要审计字段合同 | Why Audit Trace Contract

企业 AI/Agent 落地的最大风险不是"慢"或"贵"，而是**不可追责**：
- 系统出错时，无法定位是哪个环节决策错误
- 无法复盘：为什么选择了快路径、为什么回退、置信度是多少
- 合规缺失：无法向审计部门/监管机构证明"系统做了什么"

**审计字段合同**的作用：
- 把"决策链"固化成**强制字段**（不可省略）
- 让 trace 可以被自动验证（字段完整性、类型校验）
- 让回滚/熔断可以被精准触发（基于 trace 字段条件）

---

## 二、核心字段定义 | Core Fields

### 必选字段（Mandatory）
每条 trace 记录必须包含以下字段：

```typescript
interface AuditTrace {
  // 基础标识
  trace_id: string;              // 请求唯一标识（可跨系统追踪）
  timestamp: string;              // ISO 8601 格式时间戳
  
  // 路由决策
  route_decision: string;         // 路由结果：system_0_fast | system_05_resonance | system_1_deep | fallback
  confidence: number;             // 置信度（0.0-1.0）
  reason: string;                 // 路由原因：whitelist_match | resonance_hit | unknown_fallback
  
  // 预算与风险
  budget_consumed: number;        // 本次消耗的预算比例（0.0-1.0）
  risk_level: string;             // 风险等级：low | medium | high
  
  // 回滚与安全
  rollback_safe: boolean;         // 是否可安全回滚
  fallback_triggered: boolean;    // 是否触发了回退路径
  confirmation_required: boolean; // 是否需要人工确认
  
  // 可选上下文
  input_hash?: string;            // 输入摘要（脱敏）
  output_hash?: string;           // 输出摘要（脱敏）
  metadata?: Record<string, any>; // 自定义元数据
}
```

---

## 三、示例 trace（假数据）| Example Traces

### 示例 1：快路径（白名单命中）
```json
{
  "trace_id": "req_20251231_001",
  "timestamp": "2025-12-31T10:00:00Z",
  "route_decision": "system_0_fast",
  "confidence": 0.95,
  "reason": "whitelist_match",
  "budget_consumed": 0.01,
  "risk_level": "low",
  "rollback_safe": true,
  "fallback_triggered": false,
  "confirmation_required": false
}
```

### 示例 2：共振命中（中等置信）
```json
{
  "trace_id": "req_20251231_002",
  "timestamp": "2025-12-31T10:01:00Z",
  "route_decision": "system_05_resonance",
  "confidence": 0.78,
  "reason": "resonance_hit",
  "budget_consumed": 0.05,
  "risk_level": "medium",
  "rollback_safe": true,
  "fallback_triggered": false,
  "confirmation_required": false
}
```

### 示例 3：回退到深推理（低置信 + 高风险）
```json
{
  "trace_id": "req_20251231_003",
  "timestamp": "2025-12-31T10:02:00Z",
  "route_decision": "fallback",
  "confidence": 0.42,
  "reason": "low_confidence_high_risk",
  "budget_consumed": 0.20,
  "risk_level": "high",
  "rollback_safe": true,
  "fallback_triggered": true,
  "confirmation_required": true
}
```

---

## 四、如何验证字段完整性 | Field Integrity Validation

### 自动校验规则
```typescript
// 伪代码示例（不含实现细节）
function validateTrace(trace: AuditTrace): ValidationResult {
  const errors = [];
  
  // 必选字段完整性
  if (!trace.trace_id) errors.push("missing trace_id");
  if (!trace.route_decision) errors.push("missing route_decision");
  
  // 类型校验
  if (typeof trace.confidence !== 'number' || trace.confidence < 0 || trace.confidence > 1) {
    errors.push("invalid confidence range");
  }
  
  // 逻辑一致性
  if (trace.risk_level === 'high' && !trace.confirmation_required) {
    errors.push("high risk must require confirmation");
  }
  
  return { valid: errors.length === 0, errors };
}
```

---

## 五、如何在 PoC 中使用 | How to Use in PoC

### PoC 阶段的 trace 验收标准
- **字段完整性**：所有必选字段 100% 覆盖
- **trace 覆盖率**：≥ 95% 的路由决策都有 trace
- **可追责测试**：给定 trace_id，能在 5 秒内定位完整决策链
- **回滚演练**：给定失败条件，能按 trace 触发精准回滚

### 交付物清单
- trace 日志文件（脱敏版，包含所有必选字段）
- 字段完整性校验报告（自动生成）
- trace 覆盖率报告（按链路/模块统计）
- 回滚演练记录（失败模式 + 触发条件 + 回滚结果）

---

## 六、完整 trace 在哪里？| Where Is the Full Trace?

完整 trace（含真实数据、业务上下文、可复刻配置）在**私有证据包**，可在以下场景提供：
- 30 分钟技术评审（屏幕共享演示）
- 付费 PoC 启动后（按保密协议交付）

---

## 📞 如何评审

请邮件说明你们的链路与想优化的指标，我会安排 30 分钟线上评审，屏幕共享完整证据包。

Email: chenmoke2022@gmail.com

