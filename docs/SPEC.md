# ai-wiki 仕様書 (v0)

_2026-04-19 起草。REQUIREMENTS.md v1 (D1-D7 closed) を前提に、データモデル・operation 仕様・scripts 構造・test 計画を定義._

---

## 1. ファイルシステム規約

### 1.1 vault (D1: `~/ai-wiki/`)

```
~/ai-wiki/
├── concepts/              # 概念 page (emergent node)
│   └── <slug>.md
├── entities/              # 人物・model・論文・ツール (固有名詞 page)
│   └── <slug>.md
├── sources/               # 原典 (不可侵の truth)
│   ├── arxiv-<id>.md      # arxiv 論文 (自動命名)
│   ├── blog-<slug>.md     # blog 記事 (手動命名)
│   ├── note-<YYYY-MM-DD>-<slug>.md  # user 手書きメモ
│   └── digest-<YYYY-MM-DD>-run-<N>.md  # ai-digest 取込アーカイブ
├── maps/                  # drill 用 explicit tree (v2 style)
│   └── <slug>.md
├── reps/                  # drill 回数記録 (v2 互換)
│   ├── <map-slug>.json    # map 別
│   └── _all.json          # map=* 横断時の reps
├── index.md               # master catalog (auto-generated、header commit 禁止)
├── hot-cache.md           # 直近 query context summary (auto-updated)
├── log.md                 # append-only 操作履歴
├── manifest.json          # ingest source の delta tracking
└── ignore.json            # ノイズ term 永続除外 (v2 互換)
```

### 1.2 skill 本体 (deploy 時)

```
~/.claude/skills/ai-wiki/
├── SKILL.md
├── scripts/
│   ├── .venv/             # sentence-transformers 他 (ai-digest の .venv を検討、要調整)
│   ├── vault.py           # 1.1 レイアウトの I/O
│   ├── ingest.py          # 4-stage pipeline orchestrator
│   ├── extract.py         # 概念抽出 (mcp-study term_extract 流用)
│   ├── resolve.py         # 新規 vs 既存 page merge
│   ├── schema.py          # coherence check, lint
│   ├── query.py           # hot-cache + index drill-down
│   ├── drill.py           # v2 count-based (tools_drill_v2 流用)
│   ├── coverage.py        # 機械的 gap detection
│   ├── pillars.py         # backlink 密度計算
│   ├── research.py        # web/arxiv 調査 (3-round)
│   ├── dispatcher.py      # slash command → script 振り分け
│   └── tests/
└── _dev/
    ├── IMPROVE.md         # iter history
    ├── REQUIREMENTS.md    # 本 repo から symlink or copy
    └── SPEC.md            # 本 repo から symlink or copy
```

### 1.3 slug 規約

- **英数字 + ハイフン、全小文字**: `wasserstein-gradient-flow`, `looped-transformer`
- **日本語 page は**: `concept-<英訳 or romaji>-jp.md` で別 page、または `aliases:` で統合 (優先)
- **同一概念の複数 alias** は YAML `aliases` に列挙、ファイルは単一

### 1.4 環境変数 (後方互換・将来拡張)

| Env | default | 用途 |
|---|---|---|
| `AI_WIKI_ROOT` | `~/ai-wiki` | vault 位置切替 (domain 別) |
| `AI_WIKI_SKILL_DIR` | `~/.claude/skills/ai-wiki` | skill 本体 (通常固定) |

---

## 2. データモデル

### 2.1 concept page (`concepts/<slug>.md`)

```markdown
---
type: concept
slug: wasserstein-gradient-flow
aliases: [Wasserstein Gradient Flow, WGF]
tags: [algorithmic, framework]
category: [cs.LG, optimal-transport, reinforcement-learning]
provenance: extracted
arxiv_refs: [2604.14265]
created: 2026-04-19
updated: 2026-04-19
drill_eligible: true
---

# Wasserstein Gradient Flow

Policy 空間を Wasserstein 距離で geometrize した RL framework...

## 関連概念
- [[Otto calculus]]
- [[optimal transport]]
- [[policy gradient]]

## 親概念 (parent::)
parent::[[reinforcement learning]]
parent::[[gradient flow methods]]

## 子概念 (child::)
child::[[Value Gradient Flow (VGF)]]

## 出典
- [[arxiv-2604.14265]] — Value Gradient Flow (Xu et al 2026)
- [[arxiv-1808.03030]] — Policy Optimization as Wasserstein Gradient Flows (Zhang 2018)
```

### 2.2 entity page (`entities/<slug>.md`)

```markdown
---
type: entity
slug: karpathy
entity_kind: person  # person | model | lab | paper | tool
aliases: [Andrej Karpathy]
created: 2026-04-19
updated: 2026-04-19
---

# Andrej Karpathy

OpenAI founding member, Tesla ex-director, 2024 LLM Wiki paradigm 提唱者...

## 関連
- [[LLM Wiki pattern]]
- [[AutoResearch]]
```

### 2.3 source page (`sources/arxiv-<id>.md`)

```markdown
---
type: source
slug: arxiv-2604.14265
source_kind: arxiv_paper
arxiv_id: 2604.14265
title: Reinforcement Learning via Value Gradient Flow
authors: [Haoran Xu, Kaiwen Hu, Somayeh Sojoudi, Amy Zhang]
published: 2026-04-16
url: http://arxiv.org/abs/2604.14265
ingested_at: 2026-04-19
ingested_from: ai-digest-run-4
provenance: extracted
---

# arxiv:2604.14265 — Value Gradient Flow

## abstract (原文、verbatim)
...

## 抽出された概念
- [[Wasserstein Gradient Flow]]  ← primary
- [[optimal transport]]
- [[behavior regularized RL]]

## 親文献 (top-3、ai-digest 由来 or S2)
- [[arxiv-2502.17416]]
- [[arxiv-1906.04349]]
- [[arxiv-1808.03030]]
```

### 2.4 map page (`maps/<slug>.md`) — v2 互換 tree

```markdown
---
type: map
slug: ai-root
description: AI 全体の派生構造 (drill 用 view)
drill_target: true
created: 2026-04-19
updated: 2026-04-19
---

# AI

AI
├─ 機械学習
│  ├─ [[Supervised learning]]
│  ├─ [[Unsupervised learning]]
│  └─ 強化学習
│     ├─ [[policy gradient]]
│     ├─ [[Wasserstein gradient flow]]  ← concepts/ への wikilink
│     └─ [[Q-learning]]
├─ 深層学習
│  ├─ [[CNN]]
│  ├─ [[Transformer]]
│  └─ [[Looped Transformer]]
└─ 最適化理論
   └─ [[optimal transport]]
```

**重要**: map のノード文字列は `[[page-slug]]` で concepts/ 等への wikilink。tree 記法 (v2 と同じ `├─ └─ │`) は保持、drill_next は wikilink 先の slug で reps を管理。

### 2.5 manifest.json

```json
{
  "version": 1,
  "sources": {
    "arxiv-2604.14265": {
      "ingested_at": "2026-04-19T10:00:00Z",
      "sha256": "...",
      "ingested_from": "ai-digest-run-4",
      "concepts_extracted": ["wasserstein-gradient-flow", "optimal-transport"],
      "status": "resolved"
    }
  },
  "pending": [
    {"kind": "ai-digest", "path": "~/ai-digest/2026-04-26.md", "run": 7}
  ],
  "last_ingest": "2026-04-19T10:00:00Z"
}
```

### 2.6 reps (v2 互換)

```json
{
  "terms": {
    "wasserstein-gradient-flow": 3,
    "looped-transformer": 1
  }
}
```

**key は slug**、v2 の生 term からの変更点。同じ alias でも slug が一意なので counting は integrity 保持。

### 2.7 index.md (auto-generated)

```markdown
<!-- auto-generated by /wiki-ingest and /wiki-lint. Do not edit manually. -->

# Wiki Index

_Last updated: 2026-04-19 10:00:00 UTC_

## 統計
- Concepts: 42
- Entities: 18
- Sources: 35 (arxiv: 30, blog: 3, note: 2)
- Maps: 4
- Drill 対象: 42 concepts + 4 maps
- 未 drill: 15 concepts

## 柱 (backlink top 20)
1. [[reinforcement-learning]] — 14 backlinks
2. [[transformer]] — 11 backlinks
...

## 最近追加
- 2026-04-19: [[wasserstein-gradient-flow]], [[value-gradient-flow]]
...

## 孤立 page (0 backlinks)
- [[obscure-concept]] ← lint で検出
```

### 2.8 hot-cache.md (auto-updated by `/wiki-query`)

```markdown
<!-- auto-generated by /wiki-query. Rotates every N queries. -->

# Hot Cache

## 最近の query (last 10)
1. 2026-04-19 10:05: "Wasserstein RL の親文献は何?" → hits: [[wasserstein-gradient-flow]], [[arxiv-2604.14265]]

## 直近 frequently accessed pages
- [[wasserstein-gradient-flow]] — 3x in last 24h
- [[looped-transformer]] — 2x
```

### 2.9 log.md (append-only)

```markdown
# Operation Log

_Append-only. Never rewrite._

2026-04-19 09:00:00 | init | Initial vault structure
2026-04-19 10:00:00 | ingest | source=ai-digest-run-4, 5 concepts created, 2 resolved
2026-04-19 10:05:00 | query | q="..." cache_hit=true
2026-04-19 10:10:00 | drill | map=ai-root, 5 terms selected, 5 reps incremented
```

---

## 3. Operation 仕様 (slash commands)

### 3.1 `/wiki-ingest`

**入力形式:**
- 引数: `source_path_or_ref`
- 種別自動判定:
  - `~/ai-digest/*.md` → digest 取込
  - `arxiv:XXXX.XXXXX` → arxiv API 取得 + source page 作成
  - `*.md` (手動 md) → 手動 note 取込
  - `--from-digest` flag → manifest.json の pending_digest から全 Run 順次

**pipeline 4-stage (Karpathy):**

1. **Ingest**: source_kind 判定 → raw content 保存 (sources/)
2. **Extract**: term_extract.py (v2 流用) で concept candidates 抽出、embedding で synonym clustering
3. **Resolve**: 各 candidate について
   - 既存 concepts/ の page と cosine sim > 0.85 なら merge (alias 追記 + backlink 追加)
   - そうでなければ新 page 作成
4. **Schema**: index.md 更新、orphan check、backlink 整合性検証

**出力 (JSON):**
```json
{
  "source": "arxiv-2604.14265",
  "concepts_created": ["wasserstein-gradient-flow", "behavior-regularized-rl"],
  "concepts_updated": ["optimal-transport", "reinforcement-learning"],
  "entities_created": [],
  "pillars_shifted": {"reinforcement-learning": "+1 backlink"},
  "warnings": []
}
```

**重要ルール:**
- source は上書きしない (同一 arxiv_id 再 ingest は no-op + warning)
- concepts の resolve 失敗 (decision point) は user prompt → 再開は別 command
- log.md に append

### 3.2 `/wiki-query`

**入力**: 自然言語質問

**処理:**
1. hot-cache.md 先読み (cache hit 可能性)
2. index.md の backlink top 20 を優先参照
3. query と関連の高い concept pages を cosine で detect
4. 該当 page の内容 + backlinks → Claude が synthesize
5. 回答は wiki page citation 付き: "Wasserstein Gradient Flow は [[arxiv-2604.14265]] で提案された..."
6. hot-cache.md 更新

**出力**: 回答文字列 (Claude 生成、cite は wikilink 形式)

### 3.3 `/wiki-lint`

**処理:**
- Orphan page 検出 (backlinks = 0 かつ not source)
- Dead wikilink 検出 (`[[X]]` だが X.md 存在せず)
- Sibling abstraction inconsistency (maps/ 内で親子関係の spot check — 将来実装)
- Backlink sparsity (concepts で backlinks < 2 が多数 → cluster 形成不十分)

**出力**: JSON レポート
```json
{
  "orphans": ["some-concept"],
  "dead_links": [{"from": "page-a", "to": "missing-page"}],
  "sparse_concepts": ["lonely-concept"],
  "suggestions": ["Run /wiki-ingest more sources to grow backlinks"]
}
```

### 3.4 `/wiki-pillars`

**処理:**
- 全 concept page を読む
- 各 page の被 wikilink 数 (backlinks) を count
- tag / category 別 top-N 計算
- "emerging cluster": 新規 concept 5+ が同じ parent:: を持つ → 柱化の予兆

**出力:**
```json
{
  "top_20": [
    {"slug": "reinforcement-learning", "backlinks": 14, "category": "cs.LG"},
    ...
  ],
  "emerging": [
    {"slug": "optimal-transport", "recent_additions": 5}
  ],
  "decayed": [  // 1 ヶ月で新規 backlink 増なし
    {"slug": "legacy-concept", "last_backlink_added": "2026-03-01"}
  ]
}
```

### 3.5 `/wiki-drill`

**入力:** map_path or `*`、limit (default 5)

**処理** (v2 drill_next 流用):
- map_path 指定時: 該当 map の tree 走査 → wikilink 先 slug を抽出 → reps/<map-slug>.json
- `*` 指定時: concepts/ 全 page → drill_eligible=true のもの → reps/_all.json
- Count ascending でソート、alpha stable
- 上位 limit 件を選択、各 reps++、prompt 生成

**prompt format**: `「ルート / 親」の「用語」について想起されるものは？`
- map mode: tree から root/parent 抽出
- `*` mode: concept の parent:: frontmatter から

**出力:**
```json
{
  "map_path": "maps/ai-root.md",
  "total_terms": 42,
  "total_reps": 120,
  "never_done": 15,
  "problems": [
    {"term": "wasserstein-gradient-flow", "count": 1, "prompt": "「AI / 強化学習」の「wasserstein-gradient-flow」について想起されるものは？"}
  ]
}
```

### 3.6 `/wiki-status`

**出力:** dashboard
```json
{
  "vault_path": "/Users/ivymee/ai-wiki",
  "counts": {"concepts": 42, "entities": 18, "sources": 35, "maps": 4},
  "drill": {"total_terms": 46, "total_reps": 120, "never_done": 15, "avg_reps": 2.6},
  "pending_ingest": [
    {"kind": "ai-digest", "path": "...", "estimated_concepts": 10}
  ],
  "last_operations": ["/wiki-ingest (10:00)", "/wiki-drill (10:10)"]
}
```

### 3.7 `/wiki-research`

**入力:** topic

**3-round 処理 (claude-obsidian pattern):**
1. Round 1: ai-digest 流 arxiv search + S2 recommendations で初期 source 群取得
2. Round 2: Round 1 の concepts の backlink expansion (related 探索)
3. Round 3: gap 充填 — 現在 vault で弱い concepts を優先補強

**各 round で候補 source を提示 → user 承認 → ingest**

### 3.8 `/wiki-coverage`

**入力:** map_path (optional)

**処理:** v2 term_coverage を vault 構造に対応
- map 内の全 wikilink 先 slug を取得
- sources/ 全件の抽出 concepts と照合
- missing (抽出されたが map に含まれない) を返す
- single source → multi-source に拡張 (source_path=None で全 source)

---

## 4. Scripts 構造

### 4.1 vault.py — I/O primitives

```python
class Vault:
    def __init__(self, root: Path): ...
    def read_concept(self, slug: str) -> ConceptPage: ...
    def write_concept(self, page: ConceptPage) -> None: ...
    def list_concepts(self) -> list[str]: ...
    def read_source(self, slug: str) -> SourcePage: ...
    # ... entities, maps, reps, manifest, ignore, log, index, hot_cache
    def append_log(self, op: str, details: dict) -> None: ...
```

**前提:** YAML frontmatter parse は `python-frontmatter` (依存追加) or 自作 regex。stdlib では前者想定。

### 4.2 ingest.py — orchestrator

```python
def ingest(vault: Vault, source: str, **opts) -> IngestReport:
    raw = stage_ingest(source)          # 1
    candidates = stage_extract(raw)      # 2
    report = stage_resolve(vault, candidates)  # 3
    stage_schema(vault, report)         # 4
    vault.append_log("ingest", {"source": source, ...})
    return report
```

### 4.3 extract.py — concept 抽出

- 既存 `tools_term.extract_terms` を流用
- 追加: embedding ベクトル計算 (ai-digest 流用、allenai-specter)
- 出力: `[{term, count, score, embedding}]`

### 4.4 resolve.py — merge logic

```python
def resolve_candidate(vault: Vault, candidate: dict) -> ResolveAction:
    # 既存 concept と cosine sim 計算
    # sim > 0.85: merge (alias 追記)
    # 0.7 < sim <= 0.85: decision point (user prompt)
    # sim <= 0.7: 新 concept 作成
```

### 4.5 schema.py — coherence / lint

```python
def lint(vault: Vault) -> LintReport: ...
def update_index(vault: Vault) -> None: ...
```

### 4.6 drill.py — v2 継承

- `tools_drill_v2.drill_next` をほぼそのまま移植
- 変更: reps key を slug に、map wikilink 解決を追加

### 4.7 dispatcher.py — slash command ルーター

```python
COMMANDS = {
    "ingest": ingest.ingest,
    "query": query.handle,
    "lint": schema.lint,
    "pillars": pillars.compute,
    "drill": drill.drill_next,
    "status": vault.status,
    "research": research.multi_round,
    "coverage": coverage.check,
}

def main():
    # argparse: cmd name + rest as JSON body
    # Vault(AI_WIKI_ROOT 環境変数 or default ~/ai-wiki/)
    # result は stdout に JSON
```

---

## 5. SKILL.md 仕様

### 5.1 構成

```
YAML frontmatter:
  name: ai-wiki
  description: Personal AI research knowledge vault with drill. Karpathy LLM Wiki pattern + mcp-study v2 drill philosophy. Ingest arxiv papers/ai-digest output, auto-organize into concepts/entities/sources graph, drill via count-based recall.

Sections:
  # Intro (1 段落)
  ## Hard rules (採点禁止 / source 不可侵 / 決定的部分は決定的 / token 効率 etc.)
  ## Vault layout (§1.1 を要約)
  ## Scripts (venv + 2 commands user-facing)
  ## Procedure (各 operation の 1 行描写)
  ## Page format (concept/source/map の YAML 例)
  ## Known failures (dropped sources 同様)
  ## Notes for the assistant
```

### 5.2 参照方針

- Claude 実行時は SKILL.md を読む → 該当 script を call
- 複雑な詳細 (Karpathy pattern の理論など) は `_dev/REQUIREMENTS.md` に誘導
- JSON 出力の format 詳細は本 SPEC.md 参照

---

## 6. テスト計画

### 6.1 pytest (offline)

| ファイル | テスト対象 |
|---|---|
| `test_vault.py` | I/O primitives (frontmatter, wikilink parse, slug gen) |
| `test_extract.py` | v2 継承部分 + embedding dedup |
| `test_resolve.py` | sim > 0.85 merge, sim < 0.7 new, middle = decision |
| `test_schema.py` | orphan / dead link 検出 |
| `test_drill.py` | count ascending, prompt format, reps persistence |
| `test_pillars.py` | backlink 集計 |
| `test_ingest_pipeline.py` | mock 4-stage の integration |

### 6.2 Fresh-Claude check (on-demand)

- Subagent に SKILL.md 渡して cold execute
- 5 Core operations (ingest/query/drill/lint/pillars) を尿路動作確認
- Output に照らして SKILL.md 欠陥洗い出し

---

## 7. 依存関係

### 7.1 Python packages

| 用途 | package | 備考 |
|---|---|---|
| frontmatter parse | `python-frontmatter` | 必須 |
| 形態素解析 (v2 継承) | `sudachipy` + 辞書 | 必須 |
| embedding | `sentence-transformers` + `allenai-specter` | ai-digest の venv 流用検討 |
| 数値計算 | `numpy` (embedding cosine) | `sentence-transformers` 依存 |
| テスト | `pytest` | 既 installed |

### 7.2 venv 戦略

**判断:** ai-digest の venv を流用すべきか否か。

- 流用案: skill 間で重複 install 避ける (torch ~2GB を共有)
- 独立案: skill を別々に更新可能、dep conflict 回避

**仮決定 (実装時再確認):** 独立 venv を `~/.claude/skills/ai-wiki/scripts/.venv/`。sentence-transformers は embedding cache を ai-digest と共有 (`~/.cache/ai-digest/embeddings.json` を読めるように path 設定)。

---

## 8. 実装 stage (REQUIREMENTS §8 の詳細化)

### S1: Vault I/O + Ingest skeleton (1 week 想定)

- `vault.py` の I/O primitives
- `sources/` への raw ingest (stage 1 のみ実装、2-4 は stub)
- manifest.json 更新
- pytest: 基本 I/O
- **完了基準**: arxiv_id 1件を `~/ai-wiki/sources/arxiv-XXX.md` に保存できる

### S2: v2 drill / coverage / ignore 移植 (3 days)

- `drill.py`, `coverage.py` を mcp-study から移植
- reps key を slug 化
- pytest: v2 tests の再利用
- **完了基準**: 手動で作った map から drill が動く

### S3: Extract + Resolve (Karpathy 真骨頂)

- extract.py (v2 + embedding)
- resolve.py (merge / new / decision)
- decision point UX 設計
- pytest: mock 候補での resolve decision
- **完了基準**: source 1件から concepts/ に 5-10 page が自動生成

### S4: Schema / Lint / Pillars

- schema.py, pillars.py
- index.md auto-update
- **完了基準**: `/wiki-lint` が orphan 検出、`/wiki-pillars` が top-20 返却

### S5: Hot-cache + Query

- query.py
- hot-cache.md rotation
- **完了基準**: 自然言語質問 → cite 付き回答

### S6: ai-digest 統合

- digest 読取 parser
- pending_digest キュー
- `/wiki-ingest --from-digest`
- **完了基準**: digest Run N → auto で 10+ concepts 生成

### S7: Research autonomous

- research.py (3-round pipeline)
- ai-digest fetch_* 関数の流用
- **完了基準**: topic を与えて新 source 取込 → vault 拡張

### S8: Fresh-Claude test + docs 固定化

- SKILL.md 最終化
- REQUIREMENTS / SPEC を `_dev/` に copy
- Subagent cold check
- **完了基準**: 別 chat で `/wiki-*` が完走

### Dependencies

```
S1 ─┬─ S2 ────────┐
    │             │
    ├─ S3 ─┬─ S4 ─┤
    │     │      │
    │     └─ S5 ─┤
    │            │
    ├─ S6 ───────┤
    │            │
    └─ S7 ───────┤
                 │
                 S8
```

---

## 9. 将来拡張 (現時点 scope 外)

- Domain 切替 (urban-wiki 等): env の `AI_WIKI_ROOT` 切替で原理的可能、skill は generic 化
- Obsidian plugin 形式: 現状は skill + vault の 2 方向、plugin 化で Obsidian UI 統合
- Multi-user / sync: git 経由で可能だが、conflict resolution は課題
- Mermaid graph 自動生成: `/wiki-pillars` の拡張として concept cluster visualization

---

_次更新: S1 実装開始時に IMPROVE.md iter-0 記録、SPEC v1 で実装中の詳細確定事項反映._
