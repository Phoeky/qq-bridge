# 记忆（模拟人类记忆）

> 给 QQ 场景的 AI 加一套「会记、会忘、会挑」的记忆，替代「只有上下文 + 三条便签」的状态。
> 实现是**独立插件**：`src/memory/`，与 `social-v2.json` 的旧便签互不覆盖，可在控制台一键开关。

---

## 1. 它解决什么

| 原来的问题 | 现在的机制 |
| --- | --- |
| 长会话 / 会话退役后「忘了聊过什么」 | 事件流按会话落盘（`state/memory/`），退役不清记忆 |
| 闲聊与重要信息一视同仁，要么全记要么全丢 | salience 打分（0~1），够显著才进长期记忆 |
| 记下来的东西不会淡忘，越积越多 | 双组分指数遗忘曲线 + 按保留率分档降采样 |
| 注入的记忆与当前话题无关 | 用最近消息做 cue 检索（中文 2-gram BM25），按「相关 × 权重」排序 |
| 人设只给一个角色名，长会话里漂成通用助手 | 每轮注入 ≤400 字**人设锚**（身份 / 说话方式 / 雷点） |

---

## 2. 控制台开关

控制台 → 左侧「聊天行为 → 记忆」：

- **启用记忆插件**：写入 `config.json` 的 `memory.enabled`，即时生效、无需重启。
  关闭后：不再注入记忆块、不再写新事件，**已有数据保留在磁盘**；唤醒提示自动回落到原来的便签记忆
  （进行中的话题 / 想说未说的话 / 对群友的印象）。
- **记忆注入预算 / 人设锚上限**：热更新 `memory.budgetChars`、`memory.anchorMaxChars`。
- **清空全部记忆**：只删 `state/memory/` 下的文件，不碰会话映射与旧便签，不可撤销。

接口（仅控制台令牌可访问；带 `x-agent-token` 的 QQ 侧 AI 会被 401/403 拒绝）：

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| GET | `/api/memory` | 开关状态、参数、统计、按会话明细 |
| POST | `/api/memory/enabled` | `{ enabled: true|false }` |
| POST | `/api/memory/config` | `{ budgetChars?, anchorMaxChars?, eventThreshold?, recentDays? }` |
| POST | `/api/memory/reset` | `{}` 清空全部；`{ key: "group:123" }` 清空单个会话 |

---

## 3. 配置

`config.json`（可整块省略，走默认值）：

```json
{
  "memory": {
    "enabled": true,
    "dir": "state/memory",
    "budgetChars": 1200,
    "anchorMaxChars": 400,
    "eventThreshold": 0.45,
    "recentDays": 7,
    "recallMinW": 0.35,
    "maxEventsRead": 5000,
    "pruneIntervalMs": 21600000,
    "migrateSocialV2": true,
    "weights": { "importance": 0.3, "confidence": 0.2, "source": 0.15, "recency": 0.15, "access": 0.1, "relation": 0.1 },
    "forgetting": { "tauFastDays": 1.5, "tauSlowDays": 30, "fastShare": 0.6 }
  }
}
```

---

## 4. 机制速览

- **编码**：`salience = 0.30·直接对我 + 0.20·情绪 + 0.20·新颖 + 0.15·关系 + 0.10·重复 + 0.05·时间感`，
  ≥ `eventThreshold` 才写入事件流；owner 发言 / 被引用 / 拍一拍为强制记忆。不调模型。
- **遗忘**：`R(Δ) = λ·e^(-Δ/τ_fast) + (1-λ)·e^(-Δ/τ_slow)`，
  `τ_fast = 1.5天 × importance`，`τ_slow = 30天 × importance × (1 + 0.5·ln(1+accessCount))`，`λ = 0.6`。
  被检索命中 = 一次回忆（间隔越大强化越多）；`R` 低到阈值后先降采样、再归档/删除。
- **权重**：`w = clamp01(0.30·importance + 0.20·confidence + 0.15·source + 0.15·recency + 0.10·access + 0.10·relation) × kind系数`。
  `source`：owner 1.0 / 自己观察 0.7 / 群友声称 0.35。**权重只在 node 侧算，模型不能写**。
- **检索**：`score = 0.60·BM25相关 + 0.40·w`，按预算切成【最近的记忆】【相关的回忆】【关于他们】。
- **成本**：唤醒注入 ≤ `budgetChars`；全程规则计算，无额外 API 调用。

---

## 5. 数据布局

```
state/memory/<会话 key 冒号转横线>/
  events-YYYY-MM-DD.jsonl   追加写的事件（只增不改）
  profiles.json             群友画像（次数/最近出现/最后一句）
  anchor.json               人设锚缓存（按角色卡 mtime+size 失效）
  self.json                 自我状态（占位，后续用）
  meta.json                 版本 / 清理时间 / 迁移标记 / 原始会话 key
```

- 结构化文件原子写（tmp + rename），覆盖前留 `.bak`。
- 首次渲染时从 `social-v2.json` 幂等迁移旧便签（`meta.migratedFromSocialV2` 标记，只做一次）。
- `retireSession` 只做 checkpoint（落盘 + 清理），**不清记忆**；只有控制台重置 / 清空才删。

---

## 6. 运维与排查

```bash
npm run test:memory      # 21 项：分词/BM25、遗忘曲线、权重、端到端编码→重载→检索、开关、汇总
node scripts/probe-memory.mjs            # 只读真实 state/social-v2.json，展示迁移/人设锚/注入块
node scripts/probe-memory.mjs group:123  # 指定会话
```

- 想让 AI「重新认识」某个群：控制台清空全部记忆，或 `POST /api/memory/reset {key}`。
- 觉得记得太碎：调高 `eventThreshold`（例如 0.55）；觉得忘得太快：调大 `forgetting.tauSlowDays`。
- 关掉后仍能从旧便签看到「进行中的话题」，说明回落正常。

---

## 7. 如何移除

1. 控制台关掉开关（写入 `memory.enabled: false`）；
2. 删除 `src/memory/`、`scripts/test-memory.mjs`、`scripts/probe-memory.mjs`；
3. 删除 `bridge.js` 里 3 处接入（`createMemoryPlugin` 导入与实例、`memoryPlugin?.encode/render/checkpoint/reset` 调用）
   与 `/api/memory*` 路由、`config.json` 的 `memory` 段、`public/console.html` 的记忆页。

因为桥接对记忆的依赖只在一个注入点（唤醒提示）和一组可选调用上，删除后其余功能不受影响。
