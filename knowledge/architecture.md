# Orelia 4リポジトリの全体像

各リポジトリの詳細は、まずそのリポジトリ自身の `CLAUDE.md` を読むこと（このファイルは横断的な要約であり、詳細を再現するものではない）。ここでは「4つがどう繋がっているか」だけをまとめる。

## リポジトリ一覧

| リポジトリ | 役割 | orelia-coreとの関係 |
|---|---|---|
| `orelia-core` | メインのRPGプラグイン本体（Item/Skill/Job/Quest/Guildなど37モジュールを1jarに統合） | — |
| `orelia-debug` | 管理者向けテストプレイ・デバッグツール（`/oladmin debug ...`系） | **hard dependency**（`plugin.yml: depend`）。orelia-coreの`ServicesManager`公開APIを直接呼ぶ |
| `orelia-serverutil` | ゲーム非依存のサーバー運用/UXプラグイン（hub転送、スコアボード、tab-list、chatなど） | **soft dependency**（`plugin.yml: softdepend`）。無くても単体で動く必要がある |
| `orelia-docs` | MkDocs Materialのプレイヤー・運営者向けドキュメントサイト | コード無し。3リポジトリのソースを読んで書く「派生物」 |

## orelia-core内部: モジュール登録順序

`OreliaPlugin#onEnable()` が唯一のエントリポイント。以下の順で `RpgModule` を登録する（登録順＝enable順、disableは逆順）:

```
[foundation: 旧orelia-core]
Database → Region → Status → Job → Gathering → Item → Skill → Effect → Economy →
Accessory → Town → Monster → Boss → Gui → Api

[content: 旧orelia-world]
→ Dialogue → Story → Event → CutScene → Quest → Dungeon → Npc → PlayerInfo → WorldApi

[social/economy: 旧orelia-extra]
→ Party → Friend → Guild → Chat → Trade → Mail → Auction → Housing → Pet → Mount →
Ranking → Achievement → ExtraApi（常に最後）
```

各ブロックの`*Api`モジュール（`Api`/`WorldApi`/`ExtraApi`）はそのブロックの最後に登録される。あるモジュールは自分より**前に登録された**モジュール／`*Api`しか参照できない（後ろのブロックの`*Api`は見えない）。

順序がアルファベット順でない箇所には必ず理由がある（例: `Accessory`は`Economy`の後ろ＝レリック強化費用がVaultの`Economy`を要る、`Town`は`Monster`の直前＝モンスター湧き抑制が`TownDetectionService`を要る）。新しいモジュールを挿すときは、この「順序の理由」を継承元のコメントとして残す。

## `orelia-debug` から見た orelia-core

- `OreliaDebugPlugin#onEnable`が起動時に`ServicesManager`から`AdminCommandRegistry`と各種`*Api`（`DebugApi`/`GuiApi`/`EconomyApi`/`StatusApi`など）を取得する。**これらのサービスが見つからなければプラグイン自身を無効化する**（hard dependencyなので中途半端に動かさない）。
- `WorldDebugApi`/`ExtraDebugApi`はsoft dependency扱い（`null`になり得る）。呼び出し側は必ずnullガードして「未インストール」と返す。
- 新しいdebugサブコマンドを追加する定型手順は`orelia-debug/CLAUDE.md`の「Adding a new debug subcommand」節に明記されている。

## `orelia-serverutil` から見た orelia-core

- OreliaCoreは**soft dependency**。`CoreIntegrationModule`が常に最後に登録され、`ScoreboardApi`/`TabListApi`/`BelownameApi`/`ChatApi`（自分自身のモジュール群）とOreliaCoreの`StatusApi`/`EconomyApi`/`JobApi`をまとめて橋渡しする。
- OreliaCoreが無い環境でも全機能が動く必要があるため、`rpg.core.*`を直接importせず、`ConfigManager`/`MessageManager`/`ColorUtil`相当を独自コピーで持つ（[`orelia-conventions`スキル](../skills/orelia-conventions/SKILL.md)参照）。

## `orelia-docs` の立ち位置

- コードは書かない。3リポジトリの実装（特にコマンドクラスの`onCommand`ロジックと`messages.yml`）を読んでプレイヤー・運営者向けに書き直す。
- 実装詳細（クラス名、メソッドシグネチャ、SQLスキーマ、ダメージ計算式そのもの）は書かない。実装と食い違うページを見つけたら、ソースを正として直す。

## 開発フロー上の注意

- `orelia-debug`/`orelia-serverutil`は`orelia-core`をjitpack経由（GitHub直参照）で取得する。並行開発時は`orelia-core`側で`publishToMavenLocal`し、依存側の一時的な`mavenLocal()`エントリで拾う（本番リリース前に消す前提、指示なく消さない）。
- コミット時はREADME.md（英語版があればREADME_EN.mdも）を更新する規約が各リポジトリにある。
