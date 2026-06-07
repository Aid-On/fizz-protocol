# fizz-protocol

**Fizz** — AITuber system in [Almide](https://github.com/almide/almide) — の共有ワイヤ契約。

すべての Fizz 部品 (source / brain / voice / avatar / hub / overlay) は
このパッケージの型にだけ依存して通信する。部品同士は直接依存しない。
[openaituber](https://github.com/Aid-On/openaituber) の
`docs/almide-component-breakdown.md` §1 で定義した契約のうち、
**ランタイムのワイヤ側**を担う (persona のオンディスク形式は
[fizz-persona](https://github.com/Aid-On/fizz-persona))。

## Install

```toml
# almide.toml
[dependencies]
fizz_protocol = { git = "https://github.com/Aid-On/fizz-protocol", tag = "v0.2.0" }
```

## Modules

| module | contract |
|---|---|
| `fizz_protocol.emotion` | 感情語彙 (正規 21 種 + エイリアス正規化) とインラインタグ文法 `[happy]` / `[fx:nod]` / `[hair:short]` |
| `fizz_protocol.comment` | 正規化コメント `Comment` — 全チャットソース (YouTube / Twitch / X / 手入力) の出力型。NDJSON ヘルパ込み |
| `fizz_protocol.speak` | `SentenceFragment` (brain 出力) / `SpeakItem` (voice 入力) |
| `fizz_protocol.command` | ワイヤコマンド — `Avatar` / `Speech` / `Scene` / `Overlay` / `Sync` の 5 ドメイン + `Response` |
| `fizz_protocol.memory` | 記憶イベント / ユーザ / セッション / `RecallResult` |

## Wire format

**手書きコーデックが契約の正。** フラットな JSON で、Option フィールドは
none のときキー省略、enum は文字列表現:

```json
{"domain":"avatar","type":"setEmotion","name":"happy","weight":0.8}
{"domain":"overlay","type":"clearBroadcast"}
{"id":"yt-1","user":"viewer","text":"こんにちは","source":"youtube"}
```

```almide
import fizz_protocol.command

let wire = "{\"domain\":\"speech\",\"type\":\"mouthOpen\",\"value\":0.5}"
let c = command.command_from_json(wire)!
let back = command.command_to_json(c)
```

> **deriving Codec を使わない理由**: almide v0.25 時点で、ペイロード付き
> variant の derived `decode` が未実装 (`"variant X payload decode not yet
> implemented"`)、また submodule の derived codec はネスト record /
> Option[record] で codegen が壊れる。契約リポジトリとして安定を最優先し、
> 全コーデックを手書きで固定している (roundtrip テストが形式の定義)。

### openaituber からの主な変更

- 62 種フラット union → ドメイン別 5 変種 (`Avatar` / `Speech` / `Scene` /
  `Overlay` / `Sync`)。ワイヤ上は `domain` + `type` の 2 キー。
  受信側は自ドメインだけ match すればよい。hub は不透明に fan-out する。
- `wave` コマンド (deprecated) を削除。`playProc` 系で代替。
- `setting.value: unknown` → `valueJson` (JSON テキストの文字列)。
- 感情語彙を基本 12 + 拡張 9 の正規 21 種に統合。エイリアス
  (`embarrassed` → `Shy` 等) は `emotion_from_tag` が正規化する。
- `memory.tags` は CSV 文字列 → `List[String]`。memory 系のフィールド名は
  旧 /api/memory レスポンスを踏襲 (event/user は snake_case、統計系は camelCase)。

## Versioning

契約リポジトリのため、breaking change は必ずバージョンを上げ git tag を打つ
(0.x 系は minor が major 扱い)。部品側は tag 固定で依存する。

| tag | 内容 |
|---|---|
| v0.2.0 | 手書きコーデック化 + フラットワイヤ形式 (derived Codec 廃止)。`Ack.ok` → `Ack.accepted` |
| v0.1.1 | comment NDJSON ヘルパ追加 |
| v0.1.0 | 初版 (derived Codec — almide 未実装機能に依存していたため動作せず) |

## Tests

```bash
almide test   # 36 tests
```

各モジュール末尾の in-file テスト (roundtrip = 形式の定義) と
`spec/protocol_test.almd` (モジュール境界越しの消費テスト) が走る。
