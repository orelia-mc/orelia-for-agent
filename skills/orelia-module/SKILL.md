---
name: orelia-module
description: orelia-coreに新しいRpgModule（機能モジュール、例：Item/Skill/Job/Quest/Guildなど）を追加したり既存モジュールを変更したりするときに使う。登録順序の決め方、repository/model/service/listener/command/guiの層構造、config.yml・messages.ymlの登録、SchemaOwnerによるDBテーブル管理、/ol・/oladminコマンド登録、rpg.apiでの公開方法をまとめている。
---

# Orelia モジュールの追加・変更

## 全体像

`orelia-core` は1つのjarに37モジュールを収めたRPGスイート（旧orelia-core/world/extraが統合されたもの）。`OreliaPlugin#onEnable()` が唯一のエントリポイントで、各機能を `RpgModule`（`rpg/core/module/RpgModule.java`）として固定順序で登録する。全体のアーキテクチャは `orelia-core/CLAUDE.md` に詳しい（このスキルはそこから「新しくモジュールを足すときに実際に触る箇所」を抜き出したもの）。

## `RpgModule` インターフェース

```java
public interface RpgModule {
    String getName();
    void onEnable(OreliaPlugin plugin);
    void onDisable();
    default void onReload() {}
}
```

- `onEnable`: 登録順に1回呼ばれる。典型的な中身は「configファイル登録 → repositoryロード → service/manager構築 → リスナー登録 → `/ol`か`/oladmin`へのコマンド登録」。
- `onDisable`: 登録の**逆順**で呼ばれる。
- `onReload`: 省略可（デフォルトno-op）。config再読込＋repository再構築を行うなら実装する（`ItemModule#reloadWeapons()` が手本）。`/oladmin reload` から `ModuleManager.reloadAll()` 経由で呼ばれる。

## 登録順序をどこに挿すか

`OreliaPlugin#onEnable()` の登録リストは**依存順**。あるモジュールは自分より**前に登録された**モジュールしか `ModuleManager#get(Class)` で参照できない。

- 依存が無ければ、その分類ブロック（foundation / content / social-economy）内でアルファベット順に挿す。
- 依存があるなら依存先の後ろに置き、**なぜそこに置いたかを `OreliaPlugin` かモジュール自身にコメントで残す**（例: `Accessory` は `Economy` の後ろ — レリック強化費用がVaultの`Economy`を要るため。`Town` は `Monster` の直前 — モンスター湧き抑制が `TownDetectionService` を要るため）。この「順序の理由コメント」は省略しないこと。将来別の人が並べ替えて壊す事故を防ぐのが目的。
- 各former-plugin block（foundation/content/social-economy）の `*Api` モジュール（`Api`/`WorldApi`/`ExtraApi`）はそのブロックの最後に置く。

## 標準的な層構造（パッケージ）

新規モジュールは `rpg.<feature>` 配下に次の層を作る（存在するものだけでよい）:

- `repository/` — 純粋なデータアクセス。config駆動（`*.yml`をテンプレートにパース）か、DB駆動（`SchemaOwner`実装、`DatabaseManager`のみに依存）のどちらか。Bukkitイベントやゲームロジックを持ち込まない。永続化不要なモジュール（Partyなど）は`repository`を作らず`manager/`に状態を直接持つ。
- `model/` — テンプレートやプレイヤー単位コンポーネントなどの素データ。
- `service/` か `manager/` — repositoryの上に乗るビジネスロジック。
- `listener/` — `onEnable`で登録するBukkitイベントハンドラ。
- `command/` — `/ol` か `/oladmin` の共有ディスパッチャに登録する `CommandExecutor`（独自のトップレベルBukkitコマンドは作らない）。
- `gui/` — GUI画面がある場合のみ（詳細は [`orelia-gui-screen` スキル](../orelia-gui-screen/SKILL.md)）。

他モジュールの内部クラス（manager/service/repository）へ直接手を伸ばさない。他モジュールの機能が必要なら、そのモジュールの `RpgModule` サブクラスが公開する getter 経由、または `rpg.api`/`rpg.world.api`/`rpg.extra.api` 経由にする。

## 依存モジュールの取得

```java
private <T extends RpgModule> T require(OreliaPlugin plugin, Class<T> type) {
    return plugin.getModuleManager().get(type)
            .orElseThrow(() -> new IllegalStateException("<name> module requires " + type.getSimpleName()));
}
```

- 同一jar内の必須依存は `onEnable` の冒頭でこの形で **fail-fast** に取得する。取得を遅延させない。
- 外部プラグイン（`orelia-debug` など）向けに公開したいものだけ `ServicesManager` を通す。同じjar内のモジュール同士は「まだロードされてないかも」という不確実性が無いので `ServicesManager` を経由する必要はない。
- 登録順序上どうしても後発モジュールにしか無い機能を使いたい場合（例: `DungeonModule` が `PartyApi` を見る）は **soft・null-guard** にする（`Optional` 経由、無ければ機能を静かにスキップ）。

## Config（`config.yml`/`messages.yml`/独自yml）

```java
plugin.getConfigManager().register("xxx.yml"); // 初回はjar内のデフォルトをコピー
YamlConfiguration config = plugin.getConfigManager().get("xxx.yml").get();
```

- `config.yml` と `messages.yml` は全モジュール共有の2ファイル（ドメインごとのセクションを持つ）。新しいドメイン専用設定を増やしたいときはこの2つに新セクションを足すのが基本で、専用ymlを増やすのはgui.yml/items.yml/skills.ymlのような「大きめのテンプレート集」だけ。
- **どの`*.yml`でも、トップレベルキー（または既存セクション配下のネストしたキー）を新しく足したら、そのファイル先頭の`config-version`を1つ上げること。** `ConfigMigrator`はこの数字を見て「バンドルされたデフォルトの方がバージョンが高いときだけ」既存ユーザーのファイルへ新キーをコメントごと差し込む（`config-version`を上げ忘れると、新しいキーを足したこと自体は動くが、**既に稼働中のサーバーの既存ファイルには永久にそのキーが増えない**——起動時に毎回スキャンされるわけではなく、バージョン比較でスキップされるだけなので、症状に気づきにくい）。
- **ユーザー向け文字列はすべて `messages.yml` のキーにする。** `ChatColor` や `&`コードのリテラルをコマンド/リスナーに直書きしない（`orelia-debug`のCLAUDE.mdにも明記された規約で、core側も同じ設計）。送信は `MessageManager`:
  - `messages.send(sender, "job.current", "job", displayName)` — prefix付きで送信、`{job}`をプレースホルダ置換
  - `messages.sendRaw(...)` — prefix無し（複数行リストやGUIタイトルなど）
  - `messages.sendWithSound(...)` — 送信＋効果音
  - 存在しないキーは例外を投げず `??key??` を返す（タイポがすぐ目視できる設計）。
- `onReload()` を実装する場合、`config.yml`の値をフィールドにキャッシュしているモジュールは reload 時に明示的に再読込・再構築する（`ScoreboardManager.updateSettings`のような「再構築ではなく既存オブジェクトへ値を反映」パターンがserverutilにある）。

## プレイヤー単位の状態を持つ場合（`PlayerDataComponent`）

参加中プレイヤーのクロスモジュール状態は`PlayerData`（`rpg.core.player`）が一元管理する。Core自身は各コンポーネントの中身を一切知らず、`Class`をキーにしたマップとして保持するだけ。新規モジュールがプレイヤー単位のデータ（装備・進行度など）を持つなら、以下の3点セットで乗る:

```java
public final class XxxComponent implements PlayerDataComponent {
    public UUID getOwner() { ... }
    // モジュール固有のフィールド
}

public final class XxxManager implements PlayerDataComponentLoader<XxxComponent> {
    public Class<XxxComponent> type() { return XxxComponent.class; }
    public XxxComponent loadOrCreate(UUID uuid) { return repository.loadOrCreate(uuid); } // 参加時、メインスレッド外
    public void save(XxxComponent component) { repository.save(component); }              // 退出時・定期オートセーブ
}
```

- `onEnable`で`plugin.getPlayerDataManager().registerLoader(new XxxManager(repository))`を呼んで登録する（`AccessoryEquipmentManager`/`StoryManager`/`DungeonPlayerManager`/`DialogueManager`が実例。ローダー自体がrepositoryへの薄い委譲になっているのが典型形）。
- 読み取り側は`playerDataManager.get(uuid).flatMap(data -> data.component(XxxComponent.class))`（無ければ`Optional.empty()`）か、自モジュールのローダーが必ず登録されている前提でよいなら`PlayerData#require(Class)`（未登録なら`IllegalStateException`で即死＝サイレントなnullスタットバグを防ぐ設計）。
- `loadOrCreate`/`save`は**メインスレッド外**で呼ばれる前提で書く（Bukkit API・他モジュールの状態に触らない、pureなrepository呼び出しに留める）。

## DB永続化（`SchemaOwner`）

```java
public interface SchemaOwner {
    void createSchemaIfNotExists() throws SQLException;
}
```

- DB駆動repositoryは`SchemaOwner`を実装し、`onEnable`内で自分のテーブルを`createSchemaIfNotExists()`する。`DatabaseManager`自体はスキーマを一切知らない（各repositoryが自分のテーブルの所有者）。
- `DatabaseManager`は`DatabaseModule`が唯一構築し、SQLite/MySQLどちらでも動く共有JDBC接続を提供する。

## コマンド登録

Bukkitのトップレベルコマンドは `/ol`（プレイヤー向け、`PlayerCommandRegistry`）と `/oladmin`（管理者向け、`AdminCommandRegistry`）の2つだけ。新しいコマンドは**この2つのどちらかにサブコマンドとして登録する**。`plugin.yml`に新規コマンドを追加しない。

```java
plugin.getPlayerCommandRegistry().register("job", new JobCommand(...), "職業を確認します。", "job");
```

- `CommandExecutor`＋（必要なら）`TabCompleter`を実装した小さなクラスを`command/`に置く。
- プレイヤー専用コマンドで`sender`が`Player`でない場合は`messages.send(sender, "command.player-only")`で弾く。
- タブ補完は`args.length`に応じた絞り込みを自前で書く（`JobCommand#onTabComplete`が手本、外部ライブラリは使わない）。

## `rpg.api` / `rpg.world.api` / `rpg.extra.api` への公開

`orelia-debug`のような**別プラグイン**から呼ばれる機能だけ、narrowな`*Api`/`*ApiImpl`インターフェースを追加し、`ApiModule`/`WorldApiModule`/`ExtraApiModule`（各ブロック最後）から`ServicesManager`で公開する。同じjar内のモジュール間連携にはこの層を使わなくてよい（`ModuleManager#get`で足りる）。新しい対外機能を増やすときは、既存の内部managerクラスをそのまま公開せず、必ずこのAPIインターフェース越しにする。

**金銭が絡む新機能（報酬・購入・オークション決済など）は、独自の`EconomyApi`を新設せず、Vaultの`Economy`を直接叩く。** クエスト報酬・NPCショップ・オークション決済など、既存の課金/報酬コードは全部この方針（`rpg.api.EconomyApi`はステータス系の別用途で既に存在するので、「お金を動かす＝EconomyApi」と早合点してラッパーを増やさないこと）。

## チェックリスト（新規モジュール追加時）

1. `rpg.<feature>` パッケージを作り、上記の層構造で実装する。
2. `RpgModule`を実装し、`OreliaPlugin#onEnable()`の適切な位置に登録＋順序の理由コメントを書く。
3. 独自ymlが要るなら`src/main/resources/`にデフォルトを置き、`ConfigManager.register`する。新しいトップレベル/ネストキーを足した既存ymlも含め、触った`*.yml`は`config-version`を上げる。
4. ユーザー向け文字列は全部`messages.yml`に追加する。
5. プレイヤー単位の状態が要るなら`PlayerDataComponent`＋`PlayerDataComponentLoader`を作り`registerLoader`する。
6. コマンドは`/ol`か`/oladmin`のレジストリに登録する（新規Bukkitコマンドを増やさない）。
7. 他プラグインに公開する必要があるものだけ`rpg.*.api`に足す（金銭は独自Api化せずVaultの`Economy`を直接使う）。
8. 純粋ロジックはBukkit型から切り離してJUnitでテストする（詳細は[`orelia-conventions`スキル](../orelia-conventions/SKILL.md)）。
9. README.md / README_EN.md を更新する（コミット時の規約）。

## 関連知識

- 4リポジトリ全体の関係・依存方向: [`../../knowledge/architecture.md`](../../knowledge/architecture.md)
- GUI画面を伴う場合: [`orelia-gui-screen` スキル](../orelia-gui-screen/SKILL.md)
- ダメージ計算・HP同期・武器強化が絡む場合: [`orelia-combat` スキル](../orelia-combat/SKILL.md)
- 横断的な細かい規約: [`orelia-conventions` スキル](../orelia-conventions/SKILL.md)
