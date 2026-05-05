# 自校准规则Schema

## calibration.yaml 完整示例

```yaml
# Knowledge Healer 自校准配置
# 此文件由系统自动更新，也可手动编辑
# 最后校准时间: 2026-05-05T20:00:00+08:00

version: 1

# ─── 过期检测阈值 ───────────────────────────────
staleness:
  thresholds:
    process_doc: 60      # 流程/SOP文档（天）
    tech_spec: 90        # 技术方案
    policy: 180          # 政策/制度
    meeting_notes: -1    # 会议纪要（-1=永不过期）
    onboarding: 30       # 入职指南
    product_doc: 45      # 产品文档
    default: 90          # 未分类文档
  
  weights:
    edit_time: 0.3       # 编辑时间信号权重
    reference_freq: 0.2  # 引用频率信号权重
    external_change: 0.5 # 外部变化信号权重
  
  score_thresholds:
    stale: 0.7           # ≥此分数判定过期
    sub_health: 0.5      # ≥此分数判定亚健康
  
  # 自校准历史（系统自动写入）
  calibration_history:
    - date: "2026-05-05"
      field: "thresholds.process_doc"
      old_value: 60
      new_value: 60
      reason: "初始值"

# ─── 冲突检测配置 ───────────────────────────────
contradiction:
  min_confidence: 0.6    # 最低置信度（低于此不报告）
  assertion_limit: 10    # 每篇文档最多提取的断言数
  
  severity_rules:
    p0_keywords:         # 出现这些关键词的冲突自动升P0
      - "合规"
      - "安全"
      - "审计"
      - "财务"
      - "法律"
    p1_keywords:         # 这些关键词升P1
      - "流程"
      - "权限"
      - "审批"
      - "标准"

# ─── 断链检测配置 ───────────────────────────────
broken_links:
  check_external: false  # 是否检查外部链接
  rate_limit: 5          # 每秒最大请求数
  timeout: 10            # 单链接检查超时（秒）

# ─── 漂移检测配置 ───────────────────────────────
drift:
  confidence_floor: 0.6  # 置信度地板（低于此不进报告）
  lookback_days: 30      # 群聊回溯天数
  max_keywords: 3        # 每条断言最多搜索关键词数
  min_mentions: 2        # 单人单次提及不算，需≥此数
  
  # 低置信度信号仍记录（供手动确认）
  log_low_confidence: true
  low_confidence_threshold: 0.4

# ─── 优先级评分 ────────────────────────────────
scoring:
  health_score_base: 100
  deductions:
    p0: 15               # 每个P0扣分
    p1: 5                # 每个P1扣分
    p2: 1                # 每个P2扣分
  floor: 0               # 最低分

# ─── 白名单 ──────────────────────────────────
whitelist:
  # 这些文档不参与巡检
  skip_docs: []
  # 示例:
  # - "wiki://node_abc123"  # 归档区不巡检
  # - "doc://token_xyz"     # 个人笔记不巡检
  
  # 这些空间整体跳过
  skip_spaces: []

# ─── 校准保护 ──────────────────────────────────
calibration_limits:
  max_adjustment_pct: 20    # 单次调整最大幅度（%）
  min_staleness_days: 14    # 有效期阈值最低下限（天）
  min_confidence: 0.4       # 置信度最低下限
  stable_after_n: 3         # 连续N次同方向调整后锁定
```

---

## 校准反馈格式

### 方式1：通过巡检报告的互动按钮

当notify=true时，每个P0/P1发现的通知消息末尾附带：

```
📊 反馈：
[✓ 准确] [✗ 误报] [⚡ 应更严格]
```

用户点击后，系统记录到：

```yaml
# feedback_log.yaml
- timestamp: "2026-05-05T20:30:00+08:00"
  finding_id: "staleness_doc_abc123"
  feedback: "false_positive"  # accurate / false_positive / too_lenient
  doc_type: "process_doc"
  current_threshold: 60
```

### 方式2：追踪修复行为（被动收集）

```yaml
# repair_tracking.yaml
- finding_id: "staleness_doc_abc123"
  reported_at: "2026-05-05"
  severity: "P0"
  repaired: true
  repaired_at: "2026-05-07"  # 2天后修复 → 确认为真阳性
  days_to_repair: 2

- finding_id: "staleness_doc_xyz789"  
  reported_at: "2026-05-05"
  severity: "P1"
  repaired: false
  days_since_report: 30  # 30天未处理 → 可能误报或优先级过高
```

### 方式3：手动编辑calibration.yaml

用户可直接修改上方配置文件中的任何值。系统下次巡检时自动读取。

---

## 校准算法

```python
# 伪代码：校准更新逻辑

def calibrate(feedback_log, current_rules):
    for doc_type in affected_doc_types(feedback_log):
        false_positives = count(feedback_log, doc_type, "false_positive")
        too_lenient = count(feedback_log, doc_type, "too_lenient")
        
        if false_positives > too_lenient:
            # 误报多 → 放宽阈值
            adjustment = min(0.2, false_positives * 0.05)  # 最大20%
            new_threshold = current_threshold * (1 + adjustment)
        elif too_lenient > false_positives:
            # 漏报多 → 收紧阈值
            adjustment = min(0.2, too_lenient * 0.05)
            new_threshold = current_threshold * (1 - adjustment)
        
        # 下限保护
        new_threshold = max(new_threshold, MIN_STALENESS_DAYS)
        
        # 记录校准历史
        log_calibration(doc_type, old, new, reason)
        
        # 连续3次同方向 → 标记为稳定
        if consecutive_same_direction(doc_type) >= 3:
            mark_stable(doc_type)
```

---

## State.json Schema

```json
{
  "version": 1,
  "last_run": {
    "timestamp": "2026-05-05T20:00:00+08:00",
    "mode": "full",
    "scope": "wiki://space_abc123",
    "status": "completed",
    "duration_seconds": 312,
    "docs_scanned": 156,
    "findings": {
      "staleness": {"p0": 2, "p1": 5, "p2": 5},
      "contradiction": {"p0": 1, "p1": 2, "p2": 0},
      "broken_links": {"p0": 0, "p1": 3, "p2": 5},
      "drift": {"p0": 2, "p1": 2, "p2": 0}
    },
    "health_score": 72
  },
  "resume_point": null,
  "baseline_path": "baselines/2026-05-05.json",
  "calibration_version": 1,
  "next_scheduled": "2026-05-12T20:00:00+08:00"
}
```
