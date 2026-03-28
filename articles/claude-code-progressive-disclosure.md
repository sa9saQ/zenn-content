---
title: "CLAUDE.mdが42,000トークン食ってた——Progressive Disclosureで94%削減した話"
emoji: "📦"
type: "tech"
topics: ["claudecode", "ai", "開発環境", "初心者"]
published: false
---

## 1,207行のCLAUDE.md

Cem Karacaという開発者の事例。

Claude Codeを使い始めて9ヶ月。CLAUDE.mdにルールを足し続けた結果、**1,207行、42,200トークン**になっていた。

1日20会話すると、**844,000トークン**がCLAUDE.mdの読み込みだけで消える。コンテキストウィンドウ200Kのうち**40%が設定ファイル**。残り60%で実際の作業をしている状態。

成長カーブはこう:

| 時期 | 行数 | トークン |
|------|------|---------|
| 1ヶ月目 | 150行 | 2,000 |
| 3ヶ月目 | 400行 | 8,000 |
| 6ヶ月目 | 800行 | 20,000 |
| 9ヶ月目 | 1,207行 | **42,200** |

加速度的に膨らむ。最初は「ちょっと足すだけ」。それが積み重なる。自分のスキル133個問題と同じパターン。

---

## Progressive Disclosureとは

**「必要なときに、必要なものだけ読み込む」**。それだけ。

CLAUDE.mdの全内容を毎回読ませるのではなく、3つの層に分ける:

| 層 | 内容 | 読み込みタイミング | トークン |
|---|------|------------------|---------|
| Layer 0 | CLAUDE.md（72行） | **常に** | ~1,900 |
| Layer 1 | Skills（7個） | **呼び出し時のみ** | ~750/個 |
| Layer 2 | References | **スキル内から必要時のみ** | 必要分のみ |

CLAUDE.mdには「何を知っているか」の目次だけ置く。詳細はSkillsとReferencesに分離する。

---

## Before/After

| シナリオ | Before（モノリス） | After（3層） | 削減率 |
|---------|-------------------|-------------|--------|
| CSSバグ修正 | 42,200 | 2,400 | **94%** |
| APIルート作成 | 42,200 | 3,150 | 93% |
| シークレット+コンプライアンス | 42,200 | 3,850 | 91% |
| 全7スキル同時読込 | 42,200 | 9,769 | 77% |

CSSバグの修正に、セキュリティルールもデプロイ手順もTDDプロセスも必要ない。モノリスだと全部読む。3層なら必要なスキルだけ読む。

**最悪ケース**（全スキル同時発火）でも77%削減。**通常は91-94%削減**。

---

## O(1)スケーリング

ここが一番重要な概念。

**モノリスCLAUDE.md（O(n)）:**
ルールを1つ追加するたびに、**全セッション**のベースラインコストが増える。10ルール追加したら10ルール分、全会話で余分に消費する。

**Progressive Disclosure（O(1)）:**
スキルを追加しても、descriptionのメタデータ（~100文字）しか増えない。本文はオンデマンドだから、ベースラインはほぼ変わらない。

自分がスキルを47個持っていても、1回の会話で実際に展開されるのは2-3個。残りはdescriptionだけ。

スキル予算さえ超えなければ、**スキルは何個追加してもベースコストが増えない**。前回の記事で予算を62%に収めたのは、このO(1)特性を維持するための条件だった。

---

## 具体的な実装

### Layer 0: CLAUDE.md（目次＋永続ルール）

```markdown
# プロジェクトルール

## WHY
ユーザー体験最優先のNext.jsアプリ

## WHAT
- src/app: ルーティング
- IMPORTANT: タスク開始前に関連docsを確認すること

## HOW
- ビルド: npm run build
- テスト: npm test

## Compact Instructions
When compacting, always preserve: modified file paths,
test/build status, current task context.
```

50行以下。永続ルール（禁止事項、ビルドコマンド等）とCompact Instructionsだけ。

**「IMPORTANT: タスク開始前に関連docsを確認すること」**。この1行が地味に重要。これがないとClaudeはドキュメントを自分から読まない。

### Layer 1: Skills（ワークフロー別）

```
~/.claude/skills/
├── security-scan/SKILL.md      # セキュリティチェック
├── test-architect/SKILL.md     # TDDプロセス
├── creative-web/SKILL.md       # LP制作
└── note-zenn-writer/SKILL.md   # 記事執筆
```

各スキルのdescription（130文字以下）だけが常時コンテキストに入る。SKILL.md本文は呼ばれたときだけ展開。

### Layer 2: References（詳細資料）

```
skills/security-scan/
├── SKILL.md                    # 概要+目次（~100行）
└── references/
    ├── owasp-checks.md         # OWASPチェックリスト
    └── exploit-patterns.md     # エクスプロイトDB
```

スキル本文が500行を超えそうな場合、詳細を `references/` に逃がす。1階層のみ（ネストするとClaudeが部分読みしてしまう）。

---

## トレードオフ: 79% vs 100%

正直に書く。Progressive Disclosureにはデメリットもある。

Vercelのエージェント評価によると、スキルの発火率（モデルが正しくスキルを呼び出す確率）は**79%**。CLAUDE.mdに直接埋め込むと**100%**。

21%の差。小さくない。

明示的なトリガーキーワード（「セキュリティチェックして」→ security-scan発火）なら精度は上がる。でも暗黙的な発火（コードを書いてたら自動的にlintスキルが起動する、等）は確実ではない。

だから**絶対に守りたいルール**はCLAUDE.mdに書く。**ワークフロー的な手順**はSkillsに置く。この使い分けが鍵。

---

## alexop.devの実装例

もう一つ実践例。alexop.devの構成:

```
project/
├── CLAUDE.md        # 50行、essentialのみ
├── .claude/agents/  # ドメイン専門家エージェント
├── docs/            # gotchas + 学習記録
└── .eslintrc.json   # コードスタイルはツールに任せる
```

500→130ターンの会話改善が報告されている（同じ品質でターン数が74%削減）。コードスタイルはLinterに任せ、CLAUDE.mdには書かない。スタイルの説明は消費するだけで効果がない。

---

## 自分の環境

| 指標 | 値 |
|------|-----|
| CLAUDE.md | 90行 |
| Skills | 47個（予算62%） |
| .claude/rules/ | 7ファイル |
| Progressive Disclosure | 3層構成 |

90行のCLAUDE.mdに47個のSkills。ベースラインは約2,000トークン。スキル予算問題を解決した時点で、無意識にProgressive Disclosureを実現していた。

42,200トークンの事例を見ると、自分の2,000トークンがどれだけ軽いかわかる。95%軽い。

---

## まとめ

| 問題 | 原因 | 対策 |
|------|------|------|
| CLAUDE.mdが肥大化 | 全部1ファイルに書く | 3層に分離 |
| 毎回42Kトークン消費 | モノリス = O(n) | Progressive Disclosure = O(1) |
| スキル追加でコスト増 | — | description以外はオンデマンド |
| 発火率が100%→79%に | Skillsは明示呼び出し | 絶対ルールはCLAUDE.mdに残す |

Progressive Disclosureは「フォルダに分ける」と言い換えてもいい。巨大な1ファイルを、目次+詳細に分けるだけ。それだけで94%削減。

---

*使用環境: Claude Code（Maxプラン $200/月）*
*参考: Cem Karaca "[42K Token Problem](https://medium.com/@cem.karaca/my-claude-md-was-eating-42-000-tokens-per-conversation-heres-how-i-fixed-it-85ffba809bd4)"、alexop.dev "[Progressive Disclosure](https://alexop.dev/posts/stop-bloating-your-claude-md-progressive-disclosure-ai-coding-tools/)"*
