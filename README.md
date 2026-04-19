# ai-wiki

AI research 分野の個人的 knowledge vault + Claude Code skill。
Karpathy LLM Wiki パターン + mcp-study v2 drill 哲学の融合として設計。

## Status

**Design phase (2026-04-19 開始)**。実装未着手。

## 目的

- AI 領域の専門知識を「pillar 事前決定なし」で動的に structure する
- ai-digest (週次 AI 研究 digest skill) の出力を自動取込み、concept graph として蓄積
- v2 流の count-based drill (採点なし retrieval practice) でカバレッジ保証の反復学習

## 設計思想

1. **Schema emerges** — 柱は backlink 密度で自然浮上、事前定義しない
2. **採点なし** — retrieval practice research (2025) に依拠
3. **決定的な部分は決定的に** — extraction / resolve / lint は LLM 依存最小
4. **Source が唯一の正解原本** — concepts/entities は cache、sources は不可侵

## リポジトリ構成

```
ai-wiki/
├── docs/
│   ├── REQUIREMENTS.md   要件定義書
│   ├── SPEC.md           仕様書 (WIP)
│   └── IMPROVE.md        iter 履歴 (WIP)
├── scripts/              Python 実装 (WIP)
└── tests/                pytest (WIP)
```

最終形では `~/.claude/skills/ai-wiki/` に deploy。
Vault 自体は `~/ai-wiki/` に置く予定 (Obsidian 互換)。

## 関連プロジェクト

- `ai-digest` skill — 源泉 source の供給元 (週次 AI 研究 digest)
- `mcp-study` — 継承元 (v2 drill 哲学、term extraction)

## 詳細

[`docs/REQUIREMENTS.md`](docs/REQUIREMENTS.md) 参照。
