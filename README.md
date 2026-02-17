# OpenClaw 对话和记忆优化初步方案

> 研究日期：2026-02-17  
> 研究者：Ami + 阿米狗  
> 状态：已实施并上线

---

## 背景与目标

OpenClaw 默认配置下存在两个核心痛点：
1. **对话 context 超限**会直接中断会话，体验割裂
2. **记忆搜索被禁用**（需要 OpenAI/Gemini/Voyage API key，成本高）

本方案目标：在不依赖任何外部 API key 的前提下，实现：
- 对话 context 的主动管理（不被动等爆）
- 本地向量语义搜索（完全离线）
- 跨会话记忆自动恢复

---

## 核心约束

> 我们假设暂时解决不了对话 context 上限这一概念和顶层限制（200k token 硬上限由模型决定，无法改变）。
> 因此，所有优化都是在这个硬上限内做最大化利用。

---

## 三步方案

### Step 1 — Token 优化 + 自动压缩

**问题**：context 默认无上限地增长，直到 API 报错才触发压缩，体验差。

**解决**：

| 配置项 | 值 | 作用 |
|--------|-----|------|
| `contextTokens` | `120000` | 软上限，到达后触发剪枝（低于模型 200k 硬上限，留出安全边际） |
| `contextPruning.mode` | `cache-ttl` | 在 Anthropic 缓存 TTL 到期前删除老 tool output，减少 cacheWrite 费用 |
| `contextPruning.keepLastAssistants` | `3` | 保留最近 3 条 assistant 消息，删除更早的 |
| `compaction.mode` | `safeguard` | 确保 Pi runtime 的 compaction 有足够预留空间 |
| `compaction.reserveTokensFloor` | `20000` | compaction 预留底线 20k token（防止压缩时空间不足） |

**原理**：把原来的"被动爆仓"改成"主动管理"，在 context 超限前就开始清理。

---

### Step 2 — 新会话自动恢复历史（QMD 本地向量搜索）

**问题**：`memory_search` 工具默认需要 OpenAI/Gemini/Voyage embedding API，没有 key 则完全禁用。

**解决**：安装并启用 **QMD**（Quick Markdown / 本地向量搜索引擎）

#### QMD 是什么

- GitHub: [tobi/qmd](https://github.com/tobi/qmd)
- OpenClaw 官方支持的实验性本地搜索后端
- 技术栈：BM25 + 向量（node-llama-cpp）+ reranking
- **完全本地运行**，GGUF 模型首次自动下载，无 API key 需求

#### 安装步骤

```bash
# 1. 安装 Bun（QMD 运行时）
curl -fsSL https://bun.sh/install | bash

# 2. 安装 QMD
bun install -g https://github.com/tobi/qmd

# 3. 解锁 postinstall（bun 默认阻止）
cd ~/.bun/install/global && bun pm trust --all

# 4. 补充缺失的 @types/node 并 build
cd ~/.bun/install/global/node_modules/@tobilu/qmd
bun add -d @types/node && bun run build

# 5. 验证
qmd --version  # → qmd 1.0.6
```

**注意**：`bun install -g` 安装的 qmd 包没有预编译 dist，需要手动 build TypeScript 源码。这是一个安装坑，已记录。

#### OpenClaw 配置

```json
{
  "memory": {
    "backend": "qmd",
    "citations": "auto",
    "qmd": {
      "command": "/Users/mini/.bun/bin/qmd",
      "includeDefaultMemory": true,
      "sessions": {
        "enabled": true,
        "retentionDays": 30
      },
      "update": {
        "interval": "5m",
        "debounceMs": 15000
      },
      "limits": {
        "maxResults": 6,
        "timeoutMs": 4000
      },
      "scope": {
        "default": "deny",
        "rules": [{ "action": "allow", "match": { "chatType": "direct" } }]
      }
    }
  }
}
```

**关键配置说明**：
- `backend: "qmd"` — 切换为本地引擎（替代内置 SQLite 索引）
- `sessions.enabled: true` — 自动导出 session 对话记录到 QMD 索引（实现跨会话搜索）
- `retentionDays: 30` — 保留 30 天历史
- `scope.rules` — 仅限直接私聊可搜索（不在群组里暴露私人记忆）
- `citations: "auto"` — 搜索结果自动附带来源文件路径

**效果**：新会话开始时，`memory_search` 可以检索过去 30 天的对话历史摘要和记忆文件，自动恢复上下文。

---

### Step 3 — Context 快满时自动提醒

**问题**：用户不知道 context 什么时候快满，只能被动等待中断。

**解决**：在压缩前的"软阈值"触发主动通知

#### 工作流程

```
context 使用量
    ↓
[120k 软上限] → contextPruning 开始清理老消息
    ↓
[还剩 ~5k token] → memoryFlush 触发（比 compaction 更早）
    ↓
    ├── 1. 把重要上下文写入 memory/YYYY-MM-DD.md
    └── 2. 发 Telegram 通知：⚠️ 对话快满了，建议发 /new
    ↓
[用户发 /new 开启新会话]
    ↓
[新会话] → memory_search 自动检索历史 → 无缝续接
```

#### 配置

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 5000,
          "prompt": "Context is nearing the compaction limit. Do TWO things:\n1. Write important context/decisions from this conversation to memory/YYYY-MM-DD.md (today's date).\n2. Use the message tool to send a Telegram notification to the user (target: 5588544200, channel: telegram) saying: '⚠️ 对话快满了，建议发 /new 开启新会话，我会自动从历史记忆中恢复上下文。'\nReply with NO_REPLY after completing both steps.",
          "systemPrompt": "Pre-compaction housekeeping: persist memories and notify user. Silent turn."
        }
      }
    }
  }
}
```

**原理**：
- `softThresholdTokens: 5000` — 在 Pi runtime 触发 compaction **之前** 5000 token 就介入
- `prompt` 中的双任务：写记忆 + 发通知
- `NO_REPLY` — 对话中不显示任何内容（静默操作），但 tools 仍然执行（Telegram 消息照常发）

---

## 整体架构图

```
用户对话
  │
  ▼
[OpenClaw Gateway]
  │
  ├── contextTokens: 120k → 软上限保护
  ├── contextPruning: cache-ttl → 清理老 tool output
  │
  ├── [context 剩 5k] → memoryFlush 触发
  │     ├── 写 memory/YYYY-MM-DD.md
  │     └── 发 Telegram 通知 → 用户
  │
  ├── compaction: safeguard → 自动压缩摘要（最后防线）
  │
  └── [用户 /new 开新会话]
        │
        ▼
  [新会话启动 — AGENTS.md 规则]
        ├── 自动读取 MEMORY.md（系统注入）
        ├── 自动读取 memory/今日+昨日.md（系统注入）
        └── 自动调用 memory_search("最近对话 上下文")
              │
              ▼
        QMD 本地向量索引（上限 5k tokens）
          ├── MEMORY.md（长期记忆）
          ├── memory/*.md（日志）
          └── sessions/*.jsonl（30天对话历史）
              │
              ▼
        自动恢复上下文，无缝续接
```

---

## Step 4 — 新会话自动触发历史检索（AGENTS.md 规则）

**问题**：QMD 搜索是"按需"的，新会话开始时需要主动触发，否则历史上下文不会自动加载。

**解决**：在 `AGENTS.md` 的"每次会话"规则中加入自动搜索指令。

#### 触发机制说明

| 层级 | 内容 | 加载方式 |
|------|------|----------|
| `MEMORY.md` | 长期记忆（核心事实、偏好） | 系统自动注入 context |
| `memory/YYYY-MM-DD.md` | 今日 + 昨日日志 | 系统自动注入 context |
| QMD session 历史 | 近 30 天对话记录 | **需调用 `memory_search` 触发** |

#### AGENTS.md 新增规则

```markdown
## Every Session

Before doing anything else:
1. Read SOUL.md — this is who you are
2. Read USER.md — this is who you're helping
3. Read memory/YYYY-MM-DD.md (today + yesterday) for recent context
4. **If in MAIN SESSION**: Also read MEMORY.md
5. **If in MAIN SESSION**: Run memory_search("最近对话 上下文") to retrieve
   relevant history from QMD — restore any ongoing topics or tasks automatically
```

**Token 消耗评估**：

| 场景 | 消耗 |
|------|------|
| QMD 本地搜索计算 | 0（纯本地） |
| 搜索结果注入 context | 1k–5k tokens |
| 上限（`maxInjectedChars: 20000`） | ~5000 tokens 硬上限 |

**`maxInjectedChars` 的作用**：无论搜到多少内容，最终注入 context 的字符数不超过 20000（约 5000 tokens），防止历史检索本身消耗过多 context 空间。

#### QMD limits 完整配置

```json
{
  "memory": {
    "qmd": {
      "limits": {
        "maxResults": 6,
        "maxSnippetChars": 700,
        "maxInjectedChars": 20000,
        "timeoutMs": 4000
      }
    }
  }
}
```

**效果**：开新会话后无需任何提示，我会自动检索近期历史并将相关上下文带入当前会话，实现无缝续接。

---

## 当前局限性

1. **QMD 首次搜索较慢**：需要自动下载 GGUF 模型（reranker + query expansion），约几百 MB
2. **session 索引延迟**：对话结束后才导出，不是实时的
3. **根本上限无法突破**：200k token 硬上限由 Anthropic 决定，所有优化只是更好地利用这 200k
4. **QMD build 依赖**：安装时需要手动 build TypeScript（bun 全局安装的包缺少 dist 目录，已知坑）

---

## 文件位置

| 文件 | 说明 |
|------|------|
| `~/.openclaw/openclaw.json` | 所有配置的实际存储位置 |
| `/Users/mini/clawd/MEMORY.md` | 长期记忆（每次主会话加载） |
| `/Users/mini/clawd/memory/YYYY-MM-DD.md` | 每日日志 |
| `~/.openclaw/agents/main/qmd/` | QMD 索引数据库 + 缓存 |
| `~/.openclaw/agents/main/sessions/*.jsonl` | 对话历史（被 QMD 索引）|

---

## 后续可探索方向

- **3层记忆架构**（@Ktaohzk 方案）：Fact/Belief 分层 + 衰减评分，进一步减少 token 占用
- **session memory search 实验性功能**：`memorySearch.experimental.sessionMemory: true`（内置 SQLite 版的 session 索引，与 QMD sessions 二选一）
- **混合搜索调优**：调整 `vectorWeight` / `textWeight` 比例优化检索精度
- **embedding 缓存**：`memorySearch.cache.enabled: true` 避免重复 embedding

---

*本文档由 Ami + 阿米狗共同研究整理，2026-02-17，v1.1 更新于同日，v1.2 QMD 修复记录追加于同日，v1.3 BM25 迁移记录追加于同日，v1.4 语义搜索升级方案追加于同日*

---

## 附录：QMD 调试修复全历程（v1.2）

> 记录于 2026-02-17，修复过程约 1 小时

### 背景

按照本文 Step 2 配置完 QMD 并重启 gateway 后，`memory_search` 工具仍然返回：

```json
{
  "results": [],
  "disabled": true,
  "error": "No API key found for provider \"openai\"..."
}
```

QMD 没有接管搜索，系统仍然在走旧的 embedding 路径。

---

### 问题一：`memory_search` 工具被标记为 disabled

**根因**：OpenClaw 的工具启用判断与后端配置是两条独立链路：
- `memory.backend = "qmd"` 只控制**实际搜索**走哪个引擎
- 工具本身的启用检查（`createMemorySearchTool`）还是会校验 embedding API key

两者互不感知。如果没有 OpenAI/Gemini/Voyage API key，`memory_search` 工具直接在注册阶段就被禁用，根本不会尝试 QMD。

**验证方法**：在 wrapper script 加日志 → 调用 `memory_search` → 日志为空，说明 wrapper 从未被执行。

**修法**：额外配置本地 embedding 模型，让工具判断认为"搜索可用"：

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "local",
        "local": {
          "modelPath": "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf"
        },
        "fallback": "none"
      }
    }
  }
}
```

> 注意：`local` provider 的配置只用于"激活"工具，实际搜索仍由 QMD 接管（`memory.backend = "qmd"` 优先级更高）。

---

### 问题二：`qmd query --json` 崩溃（SIGABRT）

**根因**：OpenClaw 内部调用的是 `qmd query --json`（带 reranking + 向量 + 查询扩展）。但 session 文件是完整对话记录，单个 chunk 长度超过 reranker 的上下文窗口（2062 tokens），导致：

1. JavaScript 层：`LlamaRankingContext` 抛出错误
2. Metal GPU 层：`GGML_ASSERT([rsets->data count] == 0) failed` → SIGABRT 崩溃

错误信息：
```
Error: The input lengths of some of the given documents exceed the context size.
Try to increase the context size to at least 2062 or use another model that supports longer contexts.
```

之后 QMD 进程崩溃，OpenClaw 检测到 QMD 失败 → 回退到内置 SQLite 引擎 → 内置引擎因缺少 API key → `disabled: true`。

**修法**：创建 wrapper script，把 `qmd query`（含 reranking）替换为 `qmd vsearch`（向量搜索，不含 reranking）：

```bash
# /Users/mini/clawd/scripts/qmd-wrapper.sh
#!/bin/bash
# QMD wrapper: replaces 'qmd query --json' with 'qmd vsearch --json'
# Workaround for reranker crash on long session chunks (>2062 tokens)

QMD_BIN="/Users/mini/.bun/bin/qmd"
ARGS=()

for arg in "$@"; do
  if [ "$arg" = "query" ]; then
    ARGS+=("vsearch")
  else
    ARGS+=("$arg")
  fi
done

exec "$QMD_BIN" "${ARGS[@]}"
```

然后在 OpenClaw 配置中将 `memory.qmd.command` 指向 wrapper：

```json
{
  "memory": {
    "qmd": {
      "command": "/Users/mini/clawd/scripts/qmd-wrapper.sh"
    }
  }
}
```

> `qmd vsearch` = 向量相似度搜索 + 查询扩展，无 reranking，输出相同 JSON 格式，OpenClaw 可直接消费。

---

### 问题三：GGUF 模型重复下载 + timeout

**根因**：
- GGUF 模型已下载到系统默认路径 `~/.cache/qmd/models/`
- OpenClaw 以独立 XDG 环境运行 QMD（`~/.openclaw/agents/main/qmd/xdg-cache/`）
- 两个路径不共享，每次 OpenClaw 调用 QMD 都会尝试重新下载模型
- 默认 `timeoutMs: 4000`（4秒），下载没完成就超时

**修法**：

1. 软链模型文件到 OpenClaw 的 XDG cache 目录：

```bash
mkdir -p ~/.openclaw/agents/main/qmd/xdg-cache/qmd/models
ln -sf ~/.cache/qmd/models/hf_ggml-org_embeddinggemma-300M-Q8_0.gguf \
    ~/.openclaw/agents/main/qmd/xdg-cache/qmd/models/
ln -sf ~/.cache/qmd/models/hf_ggml-org_qwen3-reranker-0.6b-q8_0.gguf \
    ~/.openclaw/agents/main/qmd/xdg-cache/qmd/models/
ln -sf ~/.cache/qmd/models/hf_tobil_qmd-query-expansion-1.7B-q4_k_m.gguf \
    ~/.openclaw/agents/main/qmd/xdg-cache/qmd/models/
```

2. 把超时时间调大：

```json
{
  "memory": {
    "qmd": {
      "limits": {
        "timeoutMs": 30000
      }
    }
  }
}
```

---

### 最终生效配置（完整版）

```json
{
  "memory": {
    "backend": "qmd",
    "citations": "auto",
    "qmd": {
      "command": "/Users/mini/clawd/scripts/qmd-wrapper.sh",
      "includeDefaultMemory": true,
      "sessions": {
        "enabled": true,
        "retentionDays": 30
      },
      "update": {
        "interval": "5m",
        "debounceMs": 15000
      },
      "limits": {
        "maxResults": 6,
        "maxSnippetChars": 700,
        "maxInjectedChars": 20000,
        "timeoutMs": 30000
      },
      "scope": {
        "default": "deny",
        "rules": [{ "action": "allow", "match": { "chatType": "direct" } }]
      }
    }
  },
  "agents": {
    "defaults": {
      "memorySearch": {
        "provider": "local",
        "local": {
          "modelPath": "hf:ggml-org/embeddinggemma-300M-GGUF/embeddinggemma-300M-Q8_0.gguf"
        },
        "fallback": "none"
      }
    }
  }
}
```

---

### 验证结果

修复后 `memory_search` 返回：

```json
{
  "results": [...],
  "provider": "qmd",
  "model": "qmd",
  "citations": "auto"
}
```

`provider: "qmd"` 确认 QMD 已接管搜索，不再走旧的 embedding 路径。

---

### 已知遗留问题

| 问题 | 说明 |
|------|------|
| vsearch 精确短语召回弱于 BM25 | 用向量搜索替代了全文搜索，对精确词组召回率稍低 |
| `qmd query`（reranking）崩溃未根治 | 根本原因是 session chunk 过长超过 reranker 上下文，暂无优雅解法 |
| `local` embedding 模型双加载 | `memorySearch.provider = "local"` 让 OpenClaw 也加载了 embedding 模型，与 QMD 冗余，未来版本可能会有更干净的 `enabled: true` 开关 |

---

### 经验教训

1. **`memory.backend = "qmd"` ≠ 工具启用**：两个配置项独立，必须同时配置 `memorySearch.provider` 才能让工具真正可用。
2. **QMD 的两个 XDG 环境完全隔离**：CLI 工具用 `~/.cache/qmd/`，OpenClaw 用 `~/.openclaw/agents/main/qmd/xdg-cache/`，模型需要手动同步。
3. **`qmd query` 不适合长文档**：session 对话记录会产生超大 chunk，直接用 reranker 会崩溃，`qmd vsearch` 是更安全的替代。
4. **OpenClaw 的降级机制会掩盖真实错误**：QMD 崩溃 → 回退内置 SQLite → 内置失败 → 显示 `disabled: true`，看起来像是"没配置"，实际是 QMD 崩了。

---

## 附录：从向量搜索迁移到 BM25（v1.3）

> 记录于 2026-02-17，基于实际测试结果

### 背景

v1.2 的方案使用 `qmd vsearch`（向量搜索）绕过了 reranker 崩溃问题，但随即发现 vsearch 本身的精确召回效果也存在问题。

### 测试结果

**测试案例**：搜索暗语关键词"自食其力大爆发"、"暗语"

| 搜索方式 | 结果 |
|----------|------|
| `qmd vsearch`（向量搜索） | 返回 session 开头片段，score 0.55~0.64，未命中目标内容 |
| `qmd search`（BM25） | 直接命中，snippet 精准显示上下文 |

直接用 SQLite FTS5 验证：
```sql
SELECT filepath, snippet(documents_fts, 2, '>>>', '<<<', '...', 60)
FROM documents_fts WHERE documents_fts MATCH '暗语' LIMIT 5;
-- 输出：sessions/0e957dfc-....md | ...记住了。 **>>>暗语<<<**：柳暗花明又一村 ...
```

### 根本原因

当前使用的本地 embedding 模型为 `embeddinggemma-300M`（仅 **3 亿参数**），向量质量严重不足：

- 对精确短语（人名、暗语、项目名）的语义距离计算基本等同于噪声
- 返回结果随机性高，无法可靠用于记忆召回
- 还需要额外加载 ~300MB 模型占用内存

### 对比分析

| 方案 | 精确召回 | 语义召回 | 资源消耗 |
|------|---------|---------|---------|
| BM25 alone | ✅ 强 | ❌ 无 | 极低（纯 SQLite FTS5） |
| 300M 向量模型 | ❌ 差 | ❌ 也差 | 高（需加载大模型） |
| BM25 + 好模型（未来） | ✅ 强 | ✅ 强 | 中（需 API 或大本地模型） |

**结论**：用 300M 模型做向量搜索，丧失了 BM25 的精确匹配优势，却没有获得任何有效的语义搜索能力。两头都不讨好。

### 变更内容

1. **`scripts/qmd-wrapper.sh`**：搜索命令从 `vsearch` 改为 `search`（BM25/FTS5）

   ```bash
   # 修改前
   ARGS+=("vsearch")
   # 修改后
   ARGS+=("search")
   ```

2. **`openclaw.json`**：禁用内置 embedding 模型加载

   ```json
   {
     "agents": {
       "defaults": {
         "memorySearch": {
           "enabled": false
         }
       }
     }
   }
   ```

### 当前遗留配置说明

`memorySearch` 下仍保留了 `provider: "local"` 等字段，但由于 `enabled: false`，实际不会加载 embedding 模型。未来清理时可完整删除 `memorySearch` 块。

### 后续方向

如果未来要恢复语义搜索能力，推荐方案：
- 接入 OpenAI `text-embedding-3-large` API
- 或使用 1B+ 参数的本地 GGUF embedding 模型
- 届时恢复混合搜索（BM25 + 向量），调整 `vectorWeight` / `textWeight`

---

## 附录：语义搜索升级方案（方案 A）（v1.4）

> 状态：**待执行**（需要 OpenAI API Key）  
> 记录于 2026-02-17

### 背景

当前系统使用纯 BM25（`qmd search`）做记忆召回，精确关键词效果好，但无法处理语义模糊查询（如：记不住原话、换了说法搜不到）。

本地 300M embedding 模型（embeddinggemma-300M）质量不足以支撑语义搜索（已验证），`qmd query`（混合搜索）因 reranker 崩溃问题无法使用。

### 方案内容

将 OpenClaw 的 memory embedding provider 切换为 OpenAI，启用高质量向量索引。

#### 所需条件

- OpenAI API Key（`sk-...`）
- 有效配额（`text-embedding-3-large` 费率：$0.13 / 百万 tokens）

#### 预计费用

个人使用场景下，MEMORY.md + memory/*.md + session 历史总量约几十万字符，每次全量重建索引消耗 < $0.01，日常增量更新接近免费。

#### 执行步骤

1. **修改 OpenClaw 配置**

```json
{
  "agents": {
    "defaults": {
      "memorySearch": {
        "enabled": true,
        "provider": "openai",
        "model": "text-embedding-3-large",
        "remote": {
          "apiKey": "sk-YOUR_KEY_HERE"
        },
        "fallback": "none"
      }
    }
  }
}
```

2. **切换 QMD wrapper 回混合搜索**

修改 `/Users/mini/clawd/scripts/qmd-wrapper.sh`，将 `search` 改回 `vsearch` 或 `query`（待测试哪种与 OpenAI embedding 配合更好）。

3. **触发索引重建**

```bash
# 手动触发 QMD 重新 embed 所有文档
XDG_CACHE_HOME=~/.openclaw/agents/main/qmd/xdg-cache \
XDG_CONFIG_HOME=~/.openclaw/agents/main/qmd/xdg-config \
/Users/mini/.bun/bin/qmd embed
```

4. **验证**

在新会话中搜索一段换了说法的内容（如用"账号合租"搜索"iShare"相关内容），确认语义命中。

#### 注意事项

- OpenClaw 的 `memorySearch` 与 `memory.backend: "qmd"` 是两套独立机制；配置后需验证 `provider` 显示为 `"openai"` 而非 `"qmd"` 才说明 embedding 生效
- QMD 的 reranker 崩溃问题（chunk > 2062 tokens）是否依然存在需重新测试；如仍崩溃，`qmd-wrapper.sh` 应保持用 `vsearch` 替代 `query`
- API Key 请勿直接写入 `openclaw.json`（会明文存储在 `~/.openclaw/`），建议使用环境变量或 gateway `env.vars` 配置
