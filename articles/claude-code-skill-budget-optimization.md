---
title: "Claude Codeのスキルを133個作ったら80個が消えていた——予算222%→62%の最適化記録"
emoji: "🔧"
type: "tech"
topics: ["claudecode", "ai", "開発環境", "最適化"]
published: true
---

## 133個作って、80個消えてた

スキルを作るのが楽しかった。

LP制作、SEO、セキュリティ、記事執筆……Claude Codeに「これもできるようにして」って教え込むたびに、自分の環境が賢くなっていく感じがした。プログラミングの知識はゼロ。でもスキルだけは133個になった。

ある日「ハーネスエンジニアリング」って言葉を見かけて、気になって `/context` を打った。

表示されたスキルを数えた。43個。

……いや待って。133個作ったよね？

残りの80個、どこいった？

---

## スキル予算という仕様

Claude Codeの Skills は `~/.claude/skills/*/SKILL.md` に配置するプロトコル定義だ。frontmatterの `description` がモデルのコンテキストに注入され、ツール呼び出し時のマッチングに使われる。

ここに**文字数上限**がある。

```
予算 ≈ context_window × 2%
デフォルト200Kウィンドウ → 約15,700文字
```

全スキルの `description長 + 109文字（XMLラッパー）` の合計が予算を超えると、**先着順で切り捨て**られる。超過分のスキルはモデルに一切見えない。呼び出されることもない。エラーも出ない。ただ消える。

この仕様は alexey-pelykh 氏の [Skill Budget Research](https://gist.github.com/alexey-pelykh/faa3c304f731d6a962efc5fa2a43abe1)（2025年12月）で初めて定量化された。

### 重要な制約

- `defer_loading` はスキルに**非適用**（MCP toolsのみ対象）
- `hidden: true` フラグは**未実装**（GitHub Issue #11045）
- `SLASH_COMMAND_TOOL_CHAR_BUDGET` 環境変数で予算を上書き可能だが、コンテキスト全体を圧迫するだけなので推奨しない

---

## 自分の環境を数値化してみたら

計測は簡単だ。

### 可視スキル数の確認

```
/context
```

Claude Codeで打つだけ。表示されるスキル一覧が「モデルから見えているスキル」。

### 総スキル数の確認

```bash
find ~/.claude/skills -name "SKILL.md" | wc -l
```

### メタデータ消費量の計算

```bash
for f in ~/.claude/skills/*/SKILL.md; do
  desc=$(sed -n '/^description:/,/^[a-z]/{ /^description:/s/.*: *//p; /^  /p }' "$f" | tr -d '\n"')
  chars=${#desc}
  total=$((chars + 109))
  name=$(basename "$(dirname "$f")")
  echo "$total $name ($chars chars desc)"
done | sort -rn
```

各スキルの消費文字数が降順で出る。合計が15,700を超えていたら予算オーバー。

### 自分の結果

| 指標 | 値 |
|------|-----|
| スキル総数 | 133 |
| メタデータ合計 | 34,987文字 |
| 予算比 | **222%（2.2倍オーバー）** |
| 可視スキル | 43（34%） |
| 不可視スキル | **80（65%）** |
| 平均description長 | 175文字 |

作ったスキルの3分の2がゴミになっていた。半年間、気づかなかった。

ショックだったのは「不調を感じなかった」こと。80個が消えてても普通に使えてた。それが逆に怖い。呼ばれるはずのスキルが呼ばれないのに、違和感がなかった。

### description長と収容数の関係

| description平均長 | 収容可能数 | 現状の適合率 |
|------------------|----------|------------|
| 263文字 | 42 | 34% |
| 200文字 | 52 | 42% |
| **130文字** | **65** | **53%** |
| 100文字 | 75 | 61% |

descriptionを短くするだけで収容数が増える。130文字が実用性とのバランスで最適解。

---

## 最適化の3ステップ

整理はClaude Code自身にやらせた。正直、自分では「どれが要る、要らない」の判断ができない。

### Step 1: アーカイブ（133→88、45個削減）

過去30日で呼び出し実績ゼロのスキルを退避した。消すんじゃなくて移動。

```bash
mkdir -p ~/.claude/skills-archive
mv ~/.claude/skills/unused-skill ~/.claude/skills-archive/
```

アーカイブ先にSKILL.mdの1行目だけ残す:

```markdown
# ARCHIVED — Absorbed into {吸収先} ({日付})
This skill has been consolidated into `~/.claude/skills/{target}/SKILL.md`.
```

45個がアーカイブ対象だった。作っただけで満足して放置してたやつが、こんなにあった。

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

吸収先のSKILL.mdにAbsorbedセクションを追加:

```markdown
## [Absorbed: red-team-audit]
攻撃者目線でのセキュリティ評価。元のred-team-auditスキルの全機能を含む。
```

同様にcodex系（8→3）、レビュー系（5→2）、UI/UX系（3→1）を統合。合計32スキルを13ターゲットに吸収した。

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
4. **英語で書く**: 日本語よりバイト効率がいい。`description`はモデル向けであって人間向けじゃない
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

正直に書くと、整理した後に劇的な変化があったかというとまだわからない。ちょっと動きが早くなった気もする。でも確信はない。

ただ、80個が見えない状態で「問題ない」と思い込んでいたのは確実に怖い。見えない不具合が一番たちが悪い。

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

---

## 学術的な裏付け

### ETH Zurich「Evaluating AGENTS.md」（2026年2月）

138リポジトリを調査した結果:

- **LLMが生成した設定ファイル** → 成功率3%低下、コスト20%増加
- **人間が作成した設定ファイル** → 成功率4%向上

情報を詰め込みすぎると、AIは大事な指示を見落とす。人間と同じだ。

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

自分の環境を確認したくなった人向け:

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

## 引き算のハーネスエンジニアリング

「もっと教え込めば賢くなる」と思ってた。逆だった。

133→47。222%→62%。やったことは足すんじゃなくて引く。それだけ。

133個作った時間は無駄だったのか。たぶん違う。作って、溢れて、消えて、整理した。この一連の失敗がなければ「スキルは数じゃない」って結論にたどり着けなかった。

でも、もっと早く気づけたらなとは思う。

---

*使用環境: Claude Code（Maxプラン $200/月）*
*参考: alexey-pelykh "[Skill Budget Research](https://gist.github.com/alexey-pelykh/faa3c304f731d6a962efc5fa2a43abe1)"、ETH Zurich "[Evaluating AGENTS.md](https://arxiv.org/abs/2502.13138)" (2026)*
