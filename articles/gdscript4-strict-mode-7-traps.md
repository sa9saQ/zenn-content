---
title: "AI コードレビュー 80件全 pass したのに、Godot 4 で parse error 13個出た話"
emoji: "🪤"
type: "tech"
topics: ["godot", "gdscript", "godotengine", "ai", "coderabbit"]
published: true
---

## TL;DR

- Godot 4.4 の個人開発で、CodeRabbit + Codex MCP のAIコードレビューを 80 件以上回して全部 pass したコードを実機起動したら、GDScript の strict mode (warning_as_error) で parse error が 13 個出てきた
- 静的レビューでは catch できない Variant 推論まわりの罠を 7 種類に整理
- 実機起動 (Godot CLI または Godot MCP Pro) が AI レビューの代替ではなく **相補** という再認識

## まず何が起きたか

個人開発で Godot 4.4 の 2D ARPG を作っている。Phase A.5 でエフェクトプリミティブ 80 種 + ポーション 30 種を実装し、PR を 8 本マージ。各 PR で CodeRabbit / Codex MCP のレビューを回し、合計 80+ 件のコメントを全部 pass させた。

それでも実機起動したらこれ。

```text
SCRIPT ERROR: Parse Error: The variable type is being inferred from a Variant value, so it will be typed as Variant. (Warning treated as error.)
          at: GDScript::reload (res://scripts/effects/ProjectilePrimitives.gd:62)
```

13 個出てきた。

しかも `godot --headless --check-only` では parse OK と判定されるのに、通常起動 (`--quit-after`) では autoload 初期化で全部エラー化する。`--check-only` と warning_as_error の適用範囲が違うのが地味に厄介。

修正に丸 1 セッション使ったので、同じ罠を踏まないように 7 種類にまとめる。

---

## 罠 1: `const PackedStringArray` は constant expression にならない

autoload の起動が通らず、依存する全 Script が compile error の連鎖を起こすパターン。

```gdscript
# ❌ コンストラクタ呼び出しは constant expression と認識されない
const WAVE2_PENDING := PackedStringArray([
    "potion_id_1", "potion_id_2",
])

# ✅ Array[String] なら literal として constant expression
const WAVE2_PENDING: Array[String] = [
    "potion_id_1", "potion_id_2",
]
```

`PackedStringArray([...])` は実行時に呼ばれるコンストラクタ。`const` は compile-time 確定の値しか受け付けないので不適。`Array[String] = [...]` は literal なので OK。

---

## 罠 2: `var x := variant_func()` の Variant 推論

戻り値型が `Variant` (例: nullable な値を null チェックで返す関数) を `:=` で受けると、left-hand side が Variant 推論されて warning_as_error。

```gdscript
static func _parse_direction(args: Dictionary) -> Variant:
    var raw = args.get("direction", null)
    if not (raw is Vector2):
        return null
    return (raw as Vector2).normalized()

# ❌ Variant 推論で warning_as_error
var direction := _parse_direction(args)
if direction == null:
    return {"ok": false}

# ✅ Variant で受けて、null チェック後に as Vector2 cast
var dir_v: Variant = _parse_direction(args)
if dir_v == null:
    return {"ok": false}
var direction: Vector2 = dir_v as Vector2
```

GDScript 4 strict mode は `var x := ...` で右辺が Variant の場合「型が決まらない」という warning を出す。これが warning_as_error 設定でエラー化。

---

## 罠 3: `weakref()` の戻り値が Variant 推論

`weakref()` の戻り値はシグネチャ上 Variant。実体は WeakRef オブジェクトだが、変数に受けるときに Variant 扱い。

```gdscript
# ❌ Variant 推論で warning_as_error
var weak_caster := weakref(caster)

# ✅ 明示型注釈 + as cast
var weak_caster: WeakRef = weakref(caster) as WeakRef
```

Object→WeakRef 変換は実装上は確定なんだけど、static type 分析的には Variant に落ちる。

---

## 罠 4: ternary `f(x) if cond else null` も Variant 推論

三項演算子の片側が `null` だと、もう片側が具体型でも Variant 推論されてしまう。

```gdscript
# ❌ Variant 推論
var c = weak_caster.get_ref() if weak_caster != null else null

# ✅ if/else 文に展開 + 型注釈
var c: Node = null
if weak_caster != null:
    var ref = weak_caster.get_ref()
    if ref != null and is_instance_valid(ref):
        c = ref
```

GDScript 4 の三項演算子は両辺の型を合一しようとして失敗 (Object と null) → Variant にフォールバック。1 行で書きたい誘惑があるが、Strict mode では if/else 文が安全。

---

## 罠 5: `clamp()` は引数依存で Variant 戻り

`clamp(x, lo, hi)` は int / float / Vector2 を受けられる generic 関数のため、戻り値型が Variant。

```gdscript
# ❌ Variant 推論
var amp := clamp(amplitude, 0.0, 100.0)
var dur := clamp(duration, 0.01, 1.0)

# ✅ float 専用 clampf を使う + 型注釈
var amp: float = clampf(amplitude, 0.0, 100.0)
var dur: float = clampf(duration, 0.01, 1.0)
```

Strict mode を有効にしているなら、`clampf()` (float) / `clampi()` (int) のように **専用版** を使うのが鉄則。`min` / `max` も同じ理屈で `minf` / `maxf` / `mini` / `maxi` を使う。

---

## 罠 6: lambda の値型キャプチャで外側変数が更新されない

これは parse error にはならない。**だが機能バグになる**。今回いちばん刺さったやつ。

GDScript 4 の lambda は外側スコープの変数を **値型** でキャプチャする (Lua / Python と同じ)。なので外側 `var` を `+=` / `-=` / `*=` しても元の変数には反映されない。

```gdscript
# ❌ 値型キャプチャ → 外側の ticks_left は永遠に減らない
var ticks_left: int = 5
timer.timeout.connect(func():
    ticks_left -= 1   # 外側に反映されない (DOT 無限ループ)
    if ticks_left <= 0:
        timer.queue_free()
)

# ✅ Dictionary は参照型 → state 経由で更新
var state := {"ticks_left": 5}
timer.timeout.connect(func():
    var n: int = (state["ticks_left"] as int) - 1
    state["ticks_left"] = n
    if n <= 0:
        timer.queue_free()
)
```

警告メッセージは `CONFUSABLE_CAPTURE_REASSIGNMENT: Reassigning lambda capture does not modify the outer local variable`。これが出たら DOT timer / 追尾弾の方向更新 / 貫通弾の damage decay みたいな「Timer 内累積処理」がほぼ確実に機能していない。

`Dictionary` を経由すれば、Dictionary 自体は参照型なので「中身を書き換える」操作が外側に反映される。

---

## 罠 7: ternary で型注釈忘れの量産バグ

罠 4 の派生だが、別パターンも頻発する。

```gdscript
# ❌ ternary で型不明 (caster が null なら -1、そうでないなら instance_id)
var caster_id: int = caster.get_instance_id() if (caster != null) else -1

# ✅ if/else に展開
var caster_id: int = -1
if caster != null and is_instance_valid(caster):
    caster_id = caster.get_instance_id()
```

書きやすいんだけど、Strict mode だと両辺型の合一が失敗するパターンが多い。1 行で書く誘惑に勝つ。

---

## なぜ AI レビューが catch できなかったか

CodeRabbit も Codex も、コードのロジック・セキュリティ・設計パターンには強い。これは間違いない。むしろ A.5-5 / A.5-7 で 80+ 件レビューを受けて、設計レベルの欠陥はかなり潰せた。

ただ以下は苦手だった。

- **言語処理系のバージョン依存挙動** — Godot 4.4 の strict mode は静的解析ツールに反映されにくい
- **`--check-only` と通常 load の差** — parser の警告レベル設定が文脈で変わる
- **lambda capture の意味論** — 値型 / 参照型の違いを「機能バグ」として認識する難易度

`var x := variant_func()` も、ロジック的には正しい。「型推論が Variant になる」のは strict mode 設定 + warning_as_error の組み合わせで初めて表面化するエラーで、AI が手元にコードだけ与えられても判断しづらい。

> 結論として、実機検証は AI レビューの「代替」ではなく「相補」。両方やる必要がある。

---

## 実機検証の最短フロー (Godot 4.4)

### CLI で parse 確認

```bash
# autoload 初期化まで含めて確認
godot --headless --quit-after 30 --path . 2>&1 | grep -E "Parse Error|Compile Error"
```

`--check-only` だけでは catch できない warning_as_error も、通常起動 (`--quit-after`) なら autoload で全部出る。WSL なら native godot バイナリを `apt install godot4` 等で入れておくと CI 用にも便利。

### Godot MCP Pro で TestRunner 起動

Claude Code から MCP 経由でテスト実行できる。

```python
mcp__godot-mcp-pro__play_scene(mode="main")
mcp__godot-mcp-pro__execute_game_script(code='''
    var result := EffectRegistryTestRunner.run_all()
    _mcp_print("pass: " + str(result.pass) + " / fail: " + str(result.fail))
''')
```

今回の修正後、`EffectRegistryTestRunner` で 102 PASS / 0 FAIL、`PotionRegistryTestRunner` で 181 PASS / 0 FAIL、合計 **283 PASS / 0 FAIL** を確認できた。

### Script Editor の auto-save 巻き戻し対策

おまけだが、Godot エディタを開いたままファイルを外部 (例: Claude Code の Edit tool / VSCode) で書き換えると、Script Editor の memory cache が auto-save で **古い内容を書き戻す** 罠がある。今回これに 30 分以上ハマった。

対策:
- 修正中は Script Editor で対象ファイルを開かない
- または検証は Godot CLI (エディタプロセスを介さない) で
- どうしても両方使うなら、エディタ再起動で memory cache をリセット

---

## まとめ

| 罠 | キーワード | 対応 |
|---|---|---|
| 1 | const PackedStringArray | `Array[String]` に書き換え |
| 2 | `var := variant_func()` | `Variant` で受けて `as` cast |
| 3 | `weakref()` 戻り | `WeakRef` 型注釈 + `as WeakRef` |
| 4 | `... if ... else null` | if/else 文に展開 |
| 5 | `clamp()` | `clampf()` / `clampi()` |
| 6 | lambda 値型 capture | Dictionary 経由で参照型化 |
| 7 | ternary 全般 | 型注釈 + if/else 文 |

AI レビュー 80+ 件全 pass しても、GDScript 4 strict mode の罠は別軸で残る。実機起動を「最後の砦」として workflow に組み込むのが現実解だった。

primitive 系の機能を AI 任せで量産している人は、`CONFUSABLE_CAPTURE_REASSIGNMENT` の警告を見たら一度全 lambda を点検する価値あり。DOT がずっと続く / homing が真っ直ぐ飛ぶ / pierce が減衰しないみたいな「気づきにくい機能バグ」がそこに潜んでる可能性が高い。

---

*検証環境: Godot 4.4-stable / WSL2 Ubuntu / CodeRabbit Pro / Codex MCP*
