# ai-wiki 要件定義書 (draft v0)

_2026-04-19 起草。Step C の集約。mcp-study v2 継承 + Karpathy LLM Wiki パターン融合を目指す._

---

## 0. 背景と動機

### 0.1 先行プロジェクト

| プロジェクト | 位置付け | 本 skill との関係 |
|---|---|---|
| `mcp-study` (v2) | 構造化学習 MCP server (東大過去問・都市計画・AI) | 設計哲学と drill 実装を継承 |
| `ai-digest` (skill) | 週次 AI 研究 digest | 源泉 (source) として結合 |
| `kg` | RAG MCP server (保険マニュアル) | 参考のみ、直接関与せず |

### 0.2 mcp-study v2 の到達点と限界

**到達点 (継承すべき):**
- 採点なし × free-recall の learning optimization (2025 research でも支持)
- 決定的 term extraction (機械的、LLM 依存回避)
- Count-based drill (100+ 問題 / 限定時間の coverage-first 学習で最適)
- source が唯一の正解原本、用語マップが index

**限界 (本 skill で乗り越えたい):**
- Tree 構造を事前決定する必要 → 「柱選定」が人間側の負担
- 定期的な柱更新 mechanism がない
- 分野 (domain) 間の概念 cross-reference なし
- ai-digest 等の動的 source 供給との結合なし

### 0.3 ユーザー (三浦義人) の真の目的

> 専門領域に入る**きっかけ (entry point)** がない。最新情報を与えられれば**芋づる式**に古典まで辿れる。

加えて構造的要求:
> 何かをパッケージとして括り、1つの抽象概念からの派生で構成された知識体系が理想。ただし「完璧に構造化」は現状困難と認識。

→ **動的に柱が emerge する growing ontology** が現実解。

---

## 1. 設計原則

### 1.1 mcp-study v2 から継承 (変更禁止)

1. **採点しない** — 想起行為自体に学習効果 (retrieval practice、2025 research で meta-analytic 支持)
2. **Source が唯一の正解原本** — 抽出は cache、source は不可侵
3. **決定的な部分は決定的に** — term 抽出・backlink 生成・dedup 等は機械的
4. **Count-based drill 優先** — coverage guarantee (SRS は 100+ 問題で starvation 問題)
5. **Token efficient** — source 1 回読込原則、hot-cache / prompt caching 活用

### 1.2 Karpathy LLM Wiki パターンから採用

1. **Schema is not fixed upfront — it emerges** — 柱を事前決定せず、backlink 密度で自然浮上
2. **LLM is the maintainer** — 定期的 refactor/lint を agent 化
3. **Markdown + wikilinks** — Obsidian 互換で人間の生 edit と graph visualization 両立
4. **Provenance marking** — extracted / inferred / ambiguous の出所を明示
5. **Delta tracking** — manifest.json で再処理最小化

### 1.3 明示的 NG (設計違反)

- LLM による回答採点 (retrieval practice の原則違反)
- Full SRS (FSRS/LECTOR 等) — coverage guarantee 崩壊
- Knowledge 本文の独立生成 (v1 の漏れの連鎖問題回帰)
- Graph-only 表現 (認知負荷、drill 困難)
- 完全自動 ontology (Path 3 は精度不足、polish コストで Path 2 と同等)

---

## 2. データモデル

### 2.1 vault 構造

```
<VAULT_ROOT>/             # D1 で決定
├─ concepts/              # 概念 page (knowledge の atom)
│  └─ *.md                # YAML frontmatter + wikilinks 本文
├─ entities/              # 人物・model・論文 (固有名詞)
│  └─ *.md
├─ sources/               # 原典 (ai-digest Core、論文、user メモ)
│  └─ arxiv-XXXX.XXXXX.md / その他
├─ maps/                  # v2 風 explicit tree (drill 用 view、optional)
│  └─ *.md
├─ reps/                  # drill 回数記録 (v2 互換、maps mirror)
│  └─ *.json
├─ index.md               # master catalog (auto-generated)
├─ hot-cache.md           # 直近 query context summary
├─ log.md                 # append-only 操作履歴
├─ manifest.json          # ingest source の delta tracking
└─ ignore.json            # ノイズ term 永続除外 (v2 互換)
```

### 2.2 Page 共通 frontmatter

```yaml
---
type: concept | entity | source
aliases: [...]            # 同義語 (embedding clustering で育てる)
tags: [...]               # 自由 tag (category と別)
category: [...]           # arxiv cat 等 (scaffold として)
provenance: extracted | inferred | ambiguous
arxiv_refs: [...]         # 関連 arxiv ID
created: YYYY-MM-DD
updated: YYYY-MM-DD
drill_eligible: true | false   # drill 対象か
---
```

### 2.3 wikilink の役割分化

| リンク種別 | 記法 | 意味 |
|---|---|---|
| concept → concept | `[[概念名]]` | 関連・類推 (primary) |
| concept → source | `[[arxiv-XXXX.XXXXX]]` | 出典 (provenance) |
| concept → entity | `[[著者名]]` | 固有名詞参照 |
| concept → concept (親子) | `parent::[[親概念]]` / `child::[[子概念]]` | tree 構造 (optional、maps 用) |

---

## 3. 操作 (slash commands)

| Command | 引数 | 処理概要 | v2 対応 |
|---|---|---|---|
| `/wiki-ingest` | source path or url | Ingest → Extract → Resolve → Schema (Karpathy 4 段階) | 拡張 (旧 map_write の進化版) |
| `/wiki-query` | 自然言語質問 | hot-cache → index → drill-down → synthesize with cites | 新規 |
| `/wiki-lint` | - | orphan / dead link / backlink sparsity 検出 | 新規 |
| `/wiki-pillars` | - | backlink 密度で top-N 柱表示 + emerging cluster 検出 | 新規 |
| `/wiki-drill` | [map_path or `*`] | count-based drill (v2 流用) | 完全継承 |
| `/wiki-status` | - | vault 統計 dashboard | 新規 |
| `/wiki-research` | topic | 3-round web/arxiv 調査 → 新 source ingest | 新規 |
| `/wiki-coverage` | [map_path] | 機械的 gap detection (v2 継承) | 継承 |

---

## 4. ai-digest 統合

```
毎週 /ai-digest 実行
  ↓ (Run N を ~/ai-digest/ に append)
/ai-digest skill 終了時に manifest.json に "pending_digest" エントリ自動追加
  ↓
user が任意タイミングで /wiki-ingest --from-digest 実行
  ↓
pending の Core 5 + Adjacent 5 を順次 ingest:
  - 各 arxiv ID → sources/arxiv-XXXX.XXXXX.md 自動生成
  - 関連キーワード → concepts/ の新規 or 既存 page に wikilink
  - 親文献 top-3 → 新 source page 候補 (opt-in で ingest)
  - 迷う配置判断のみ user 承認 point (Path 2: decision-point pattern)
```

---

## 5. スコープ

### 5.1 IN (含む)

- concepts / entities / sources / maps の vault 管理
- Ingest pipeline (manual & ai-digest 連携)
- Drill (count-based、v2 継承)
- Coverage 検証 (機械的、multi-source)
- Pillars 検出 (backlink 密度)
- Lint (orphan / dead link / sparsity)
- hot-cache による query 効率化
- Embedding ベース synonym clustering (ai-digest venv 流用)
- Provenance marking
- Delta tracking (manifest.json)

### 5.2 OUT (含まない)

- LLM 採点・回答品質判定
- Full SRS schedule
- Knowledge 本文の自動生成 (summary 以上のもの)
- 完全自動 ontology 構築
- Graph visualization (Obsidian 本体に委譲)
- MCP server 実装 (skill のみ)
- Multi-user / sync 機能

---

## 6. 開放された決定事項 (D1-D7)

### D1: vault location
- a) `~/ai-wiki/` (Obsidian 互換重視、clean)
- b) `~/Projects/ai-wiki/`
- c) `~/Projects/mcp-study/data/` 相乗り
- **推奨: a**

### D2: 既存 mcp-study data の取り扱い
- a) `data/sources/*` と `data/maps/ai/*` 全移行 (継承)
- b) 空から (clean slate)
- c) sources のみ移行、maps は Karpathy 式再構築
- **推奨: c**

### D3: ai-digest 既存 Run 1-6 の取り扱い
- a) 一括 ingest seed として投入
- b) 新規 Run から順次
- c) user 手動のみ
- **推奨: b**

### D4: skill 名と vault の関係
- a) skill: `/wiki-*`, vault: `~/ai-wiki/` (AI 専用明示)
- b) skill: `/wiki-*`, vault: `~/wiki/` (domain agnostic)
- c) skill: `/study-*`, vault: `~/study-vault/` (mcp-study 継承)
- **推奨: a**

### D5: 初期 scope の domain 制限
- a) AI 専用、後から domain 別 vault 可能設計
- b) 最初から domain 切替機構付き
- **推奨: a**

### D6: Pillars 更新頻度
- a) `/wiki-ingest` 毎に再計算 (常に最新)
- b) `/wiki-pillars` 実行時のみ
- c) 週次 auto
- **推奨: a**

### D7: Drill 対象
- a) `maps/*.md` の tree のみ (v2 compat)
- b) `concepts/*.md` 全 page (emergent)
- c) 両方 option (default map、`*` で全 concepts)
- **推奨: c**

---

## 7. 成功基準 (acceptance criteria)

1. **Coverage guarantee 維持**: drill で未回答 term が必ず出題順に達する (v2 継承テスト継承)
2. **Fresh-Claude 再現**: 別チャット / subagent で `/wiki-*` が SKILL.md だけから実行できる
3. **Obsidian 互換**: `~/ai-wiki/` を Obsidian で開くと graph view が正しく表示される
4. **ai-digest 結合動作**: 週次 digest 後に `/wiki-ingest --from-digest` で自動取込み可能
5. **Pillars emerge 検証**: 20+ source ingest 後、backlink top-10 が直感的な「AI の柱」に一致
6. **Token 効率**: 100 concepts vault で `/wiki-query` が < 10k tokens で完結 (hot-cache ヒット時)
7. **Regression test**: pytest で ingest/resolve/drill/coverage の deterministic 部分を gate

---

## 8. 実装ステージ案 (D1-D7 確定後に SPEC.md で詳細化)

| Stage | 内容 | 依存 |
|---|---|---|
| S1 | Vault I/O + Ingest pipeline skeleton | - |
| S2 | v2 drill / coverage / ignore 移植 | S1 |
| S3 | Karpathy 風 wikilink resolve + provenance | S1 |
| S4 | Pillars / Lint | S1, S3 |
| S5 | hot-cache + Query | S1, S3 |
| S6 | ai-digest 統合 | S1, S3 |
| S7 | Research (web 連携 autonomous) | S1, S5 |
| S8 | Fresh-Claude test + docs 固定化 | 全部 |

---

## 9. 参考文献

- [Karpathy LLM Wiki (a2a-mcp 解説)](https://a2a-mcp.org/blog/andrej-karpathy-llm-knowledge-bases-obsidian-wiki)
- [claude-obsidian (AgriciDaniel)](https://github.com/AgriciDaniel/claude-obsidian)
- [obsidian-wiki (Ar9av)](https://github.com/Ar9av/obsidian-wiki)
- [Retrieval Practice 2025 meta-review](https://pmc.ncbi.nlm.nih.gov/articles/PMC12292765/)
- [Claude prompt caching (90% reduction)](https://medium.com/@kuldeep.paul08/prompt-compression-techniques-reducing-context-window-costs-while-improving-llm-performance-afec1e8f1003)
- 自己作成: mcp-study `scripts/要件定義書.md` (v2 の原要件)

---

_次更新: D1-D7 user 決定後、SPEC.md 起草と合わせて本ファイル §6 を closed decisions 化._
