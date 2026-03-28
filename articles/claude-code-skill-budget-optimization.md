---
title: "Claude Codeスキル予算 222%→62% に最適化した全記録"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "ai", "開発環境", "最適化"]
published: false
---

# Claude Codeスキル予算 222%→62% に最適化した全記録

## TL;DR

- Claude Codeにはスキルメタデータの**文字数予算**がある（コンテキストの約2%、デフォルト15,700文字）
- 133スキルを運用していたら予算222%で**80スキル（65%）がモデルから不可視**だった
- 47スキルに整理して予算62%まで削減。全スキルが可視状態に
- 計算式: 各スキル消費 = `description文字数 + 109`（XMLオーバーヘッド）

---

## スキル予算とは何か

Claude Codeの Skills は `~/.claude/skills/*/SKILL.md` に配置するプロトコル定義だ。frontmatter の `description` がモデルのコンテキストに注入され、ツール呼び出し時のマッチングに使われる。

ここに**文字数上限**がある。

```
予算 ≈ context_window × 2%
デフォルト200Kウィンドウ → 約15,700文字
```

全スキルの `description長 + 109文字（XMLラッパー）` の累積が予算を超えると、**先着順で切り捨て**られる。超過分のスキルはモデルに一切見えない。呼び出されることもない。

この仕様は alexey-pelykh 氏の [Skill Budget Research](https://gist.github.com/alexey-pelykh/faa3c304f731d6a962efc5fa2a43abe1)（2025年12月）で初めて定量化された。

### 重要な制約

- `defer_loading` はスキルに**非適用**（MCP toolsのみ対象）
- `hidden: true` フラグは**未実装**（GitHub Issue #11045）
- `SLASH_COMMAND_TOOL_CHAR_BUDGET` 環境変数で予算を上書き可能だが、コンテキスト全体を圧迫するだけなので推奨しない

## 計測: 自分の環境を数値化する

### Step 1: 可視スキル数を確認

```
/context
```

Claude Code内でこのコマンドを実行する。表示されるスキル一覧が「モデルから見えているスキル」だ。

### Step 2: 総スキル数を確認

```bash
find ~/.claude/skills -name "SKILL.md" | wc -l
```

### Step 3: メタデータ消費量を計算

```bash
for f in ~/.claude/skills/*/SKILL.md; do
  desc=$(sed -n '/^description:/,/^[a-z]/{ /^description:/s/.*: *//p; /^  /p }' "$f" | tr -d '\n"')
  chars=${#desc}
  total=$((chars + 109))
  name=$(basename "$(dirname "$f")")
  echo "$total $name ($chars chars desc)"
done | sort -rn
```

各スキルの消費文字数が降順で出力される。合計が15,700を超えていたら予算オーバーだ。

### 自分の計測結果

| 指標 | 値 |
|------|-----|
| SKILL.md保有スキル | 133 |
| メタデータ合計 | 34,987文字 |
| 予算比 | **222%** |
| 可視スキル | 43（34%） |
| 不可視スキル | **80（65%）** |
| 平均description長 | 175文字 |

作ったスキルの3分の2がゴミになっていた。

### description長と収容数の関係

| description平均長 | 収容可能数 | 現状の適合率 |
|------------------|----------|------------|
| 263文字 | 42 | 34% |
| 200文字 | 52 | 42% |
| **130文字** | **65** | **53%** |
| 100文字 | 75 | 61% |

description を短くするだけで収容数が増える。130文字が実用性とのバランスで最適解。

---

## 最適化の3ステップ

### Step 1: アーカイブ（133→88、45個削減）

過去30日で呼び出し実績ゼロのスキルを `~/.claude/skills-archive/` に移動した。

```bash
mkdir -p ~/.claude/skills-archive
mv ~/.claude/skills/unused-skill ~/.claude/skills-archive/
```

アーカイブ先にSKILL.mdの1行目だけ残す:

```markdown
# ARCHIVED — Absorbed into {吸収先} ({日付})
This skill has been consolidated into `~/.claude/skills/{target}/SKILL.md`.
```

45個がアーカイブ対象だった。

### Step 2: 統合吸収（88→47、32個→13個に統合）

機能が重複するスキルを「Absorbed」パターンで統合した。

**例: セキュリティ系15+スキル → 3スキルに統合**

```
Before:
  security-scan, vulnerability-hunter, penetration-tester,
  red-team-audit, csrf-detector, idor-detector,
  supply-chain-gate, npm-audit-gate ... (15+)

After:
  security-scan（一次診断）
  vulnerability-hunter（深掘り脆弱性分析）
  penetration-tester（PoC作成 + red-team-audit吸収）
```

吸収先のSKILL.mdに以下を追加:

```markdown
## [Absorbed: red-team-audit]
攻撃者目線でのセキュリティ評価。元のred-team-auditスキルの全機能を含む。
```

同様に:
- codex-* 系（8→3に統合）
- レビュー系（5→2に統合）
- UI/UX系（3→1に統合）

合計32スキルを13ターゲットに吸収。

### Step 3: description圧縮（全スキル130文字以下）

圧縮前の最悪例:

```yaml
# 454文字
description: "Comprehensive security scanning tool that analyzes web applications,
APIs, and infrastructure for vulnerabilities including OWASP Top 10, injection attacks,
authentication flaws, and provides detailed remediation guidance with severity ratings..."
```

圧縮後:

```yaml
# 128文字
description: "Scan for OWASP Top 10 vulns in web apps and APIs. Use when: security check, vulnerability scan, pen test needed."
```

**圧縮のルール:**

1. **動詞で始める**: "Scan...", "Generate...", "Analyze..."
2. **先頭50文字にトリガーキーワードを集約**: auto_triggerの精度は先頭部分で決まる
3. **"Use when:" パターン**: 発火条件を明示する
4. **英語で書く**: 日本語よりバイト効率が良い。`description`はモデル向け、人間向けではない
5. **コロンを含む場合はダブルクォート必須**: YAMLパースエラー防止

---

## Before / After

| 指標 | Before | After | 削減率 |
|------|--------|-------|--------|
| スキル数 | 133 | 47 | -65% |
| メタデータ合計 | 34,987文字 | 9,763文字 | -72% |
| 予算使用率 | 222% | 62% | — |
| 不可視スキル | 80 | 0 | -100% |
| 平均description | 175文字 | 98文字 | -44% |

---

## 補足テクニック

### `disable-model-invocation: true`

手動でしか使わないスキル（デプロイ系、デバイス接続系）にこのフラグを設定する。

```yaml
---
name: delivery-packager
description: "Package deliverables with 44-item checklist into zip."
disable-model-invocation: true
user-invocable: true
---
```

このスキルは予算を**一切消費しない**。`/` メニューからの手動呼び出しのみ有効。`user-invocable: false`（メニュー非表示のみ）とは異なり、コンテキストからも完全に消える。

### Progressive Disclosure（500行超スキル対策）

SKILL.md本文が長いスキルは、コンテンツ自体もトークンを消費する。目次レベルに圧縮し、詳細を `references/` に分離する。

```
skills/
  penetration-tester/
    SKILL.md          # 概要+目次のみ（~100行）
    references/
      owasp-checks.md # 詳細チェックリスト
      exploit-db.md   # エクスプロイトDB
```

ClaudeFastの事例では**82%のトークン削減**が報告されている。

### PostCompact hookで文脈維持

長いセッションでcompaction（コンテキスト圧縮）が発生すると、作業文脈が失われることがある。`PostCompact` hookでサマリーをファイルに保存する:

```json
{
  "hooks": {
    "PostCompact": [{
      "type": "command",
      "command": "echo $COMPACT_SUMMARY >> .claude/compact-log.md"
    }]
  }
}
```

---

## 学術的な裏付け

### ETH Zurich「Evaluating AGENTS.md」（2026年2月）

138リポジトリを調査。結論:

- **LLMが生成した設定ファイル** → 成功率3%低下、コスト20%増加
- **人間が作成した設定ファイル** → 成功率4%向上

情報は多ければいいわけではない。質が高く量が少ない設定が最強。

### HumanLayer「Skill Issue: Harness Engineering」（2026年3月）

> 「問題はモデルではなくハーネスにある」

Opus 4.6はClaude Code内のベンチマークで33位だが、別のハーネスでは5位。同じモデルでもハーネス設計で結果が激変する。

### コミュニティの実測データ

| 構成 | CLAUDE.md | スキル数 | 効果 |
|------|----------|---------|------|
| ミニマリスト | 50-106行 | 0-5 | 1日2時間節約 |
| パワーユーザー | 100-300行 | 12-40 | 週18時間節約 |
| フレームワーク型 | 500-1000+行 | 40-100+ | 要最適化 |

okhlopkov.com の事例: CLAUDE.md 106行 + MCP 2台だけで、30ページレポートを1晩で完成。

---

## チェックリスト

自分の環境を最適化するための手順:

- [ ] `/context` で可視スキル数を確認
- [ ] 総スキル数と比較（差分 = 不可視スキル）
- [ ] description文字数を計測、合計が15,700を超えていないか
- [ ] 30日間未使用のスキルをアーカイブ
- [ ] 重複スキルを「Absorbed」パターンで統合
- [ ] 全descriptionを130文字以下に圧縮
- [ ] 手動専用スキルに `disable-model-invocation: true` を設定
- [ ] 500行超のSKILL.mdを Progressive Disclosure で分離
- [ ] 最適化後に `/context` で全スキル可視を確認

---

## 既存記事との比較

| 指標 | alexey-pelykh | polipoli記事 | shimo4228記事 | **本記事** |
|------|--------------|-------------|-------------|-----------|
| スキル数 | 63 | 引用のみ | 63 | **133** |
| 不可視数 | 21 (33%) | 引用のみ | 不明 | **80 (65%)** |
| 予算使用率 | — | — | — | **222%→62%** |
| 統合手法 | なし | なし | 棚卸し3回 | **32→13統合** |
| Before/After | 計算のみ | なし | 体験談 | **完全実測** |

日本語圏で、この規模の実測データ付き最適化ケーススタディは初だと思う。

---

## まとめ

スキルは武器だ。ただし、武器庫がパンクしたら何も取り出せない。

Claude Codeのスキル予算は15,700文字。この壁を知らずにスキルを増やし続けると、大半が消える。「足せば賢くなる」は幻想。**引けば強くなる**が正しい。

133→47。222%→62%。これが「引き算のハーネスエンジニアリング」の全記録だ。
