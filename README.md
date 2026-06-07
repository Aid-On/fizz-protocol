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
fizz_protocol = { git = "https://github.com/Aid-On/fizz-protocol", tag = "v0.1.0" }
```

## Modules

| module | contract |
|---|---|
| `fizz_protocol.emotion` | 感情語彙 (正規 21 種 + エイリアス正規化) とインラインタグ文法 `[happy]` / `[fx:nod]` / `[hair:short]` |
| `fizz_protocol.comment` | 正規化コメント `Comment` — 全チャットソース (YouTube / Twitch / X / 手入力) の出力型 |
| `fizz_protocol.speak` | `SentenceFragment` (brain 出力) / `SpeakItem` (voice 入力) |
| `fizz_protocol.command` | ワイヤコマンド — `Avatar` / `Speech` / `Scene` / `Overlay` / `Sync` の 5 ドメイン + `Response` |
| `fizz_protocol.memory` | 記憶イベント / ユーザ / セッション / `RecallResult` |

## Wire format

Almide の `deriving Codec` が生成する tagged JSON をそのまま使う:

```json
{"tag": "Overlay", "value": {"tag": "SetReply", "value": {"text": "やっほー"}}}
```

```almide
import fizz_protocol.command

let c = command.Overlay(command.SetReply { text: "やっほー" })
let wire = command.command_to_json(c)
let back = command.command_from_json(wire)   // ok(c)
```

旧 openaituber のフラット形式 (`{"type": "setReply", ...}`) とは**互換しない**。
Fizz は新契約であり、移行ブリッジが必要になった場合は変換部品を別途立てる。

### openaituber からの主な変更

- 62 種フラット union → ドメイン別 5 変種 (`Avatar` / `Speech` / `Scene` / `Overlay` / `Sync`)。
  受信側は自ドメインだけ match すればよい。hub は不透明に fan-out する。
- `wave` コマンド (deprecated) を削除。`playProc` 系で代替。
- `setting.value: unknown` → `SettingChanged.value_json: Option[String]` (JSON テキスト)。
- 感情語彙を基本 12 + 拡張 9 の正規 21 種に統合。エイリアス
  (`embarrassed` → `Shy` 等) は `emotion_from_tag` が正規化する。
- `memory.tags` は CSV 文字列 → `List[String]`。

## Versioning

契約リポジトリのため、breaking change は必ずバージョンを上げ git tag を打つ
(0.x 系は minor が major 扱い)。部品側は tag 固定で依存する。

## Tests

```bash
almide test
```

各モジュール末尾の in-file テスト (Codec roundtrip) と
`spec/protocol_test.almd` (モジュール境界越しの消費テスト) が走る。
