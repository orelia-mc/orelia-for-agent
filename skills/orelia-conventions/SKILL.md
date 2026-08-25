---
name: orelia-conventions
description: Oreliaプラグイン群（orelia-core/orelia-debug/orelia-serverutil/orelia-docs）に共通する細かいコーディング規約やスタイルを確認したいときに使う。&系カラーコードの使い分け、設計判断を残すJavadocの書き方、fail-fastとnull-guardの使い分け、テストの書き方、リポジトリ間の連携（soft-dependency、ServicesManager、mavenLocal開発ループ）をまとめている。
---

# Orelia 横断的なコーディング規約

具体的なタスク別の手順は [`orelia-gui-screen`](../orelia-gui-screen/SKILL.md)・[`orelia-module`](../orelia-module/SKILL.md) を見ること。ここに書くのは、どのタスクにも顔を出す細かい習性。

## カラーコードの使い分け

`rpg.util.ColorUtil`（orelia-serverutilは独自の軽量コピーを持つ、後述）が3系統をまとめて扱う:

| 系統 | 書き方 | 用途 |
|---|---|---|
| vanilla legacy | `&a`, `&c`, `&l` など | vanillaのチャット装飾がそのまま欲しいとき |
| hex | `&#RRGGBB` | vanillaに無い任意色が要るとき（内部で`§x§R§R...`に展開） |
| **カスタム** | `&%<char>` | **GUIのname/loreはほぼこれ**。vanillaと被らない専用パレット |

- カスタムパレットは`ColorUtil.CUSTOM_COLORS`に定義済み（`0`〜`9`+`a`〜`h`、それぞれ黒〜茶の18色）。新しい色が要るときはここに1行足すだけでよく、既存の色を上書きしない。
- 未定義の`&%<char>`は**黙って消えずにそのまま文字として残る**設計（typoがゲーム内で目視できる）。新しい文字を使う前に必ず`CUSTOM_COLORS`に登録してあるか確認する。
- 詳しい対応表: [`../../knowledge/color-codes.md`](../../knowledge/color-codes.md)
- バー表現（`&m`打ち消し線の繰り返し）は[`orelia-gui-screen`スキル](../orelia-gui-screen/SKILL.md)を参照。

## Javadoc: 「何をするか」より「なぜこうしたか」

このコードベースのクラス/メソッドコメントは、実装から読み取れる事実の言い直しではなく、**次のいずれかが書けるときに書く**:

- なぜこの設計にしたか（他の選択肢を検討して却下した理由）
- 一見自明に見えて実は罠がある挙動（例: 1tick遅延させないとロールバックに上書きされる、`setHealth`を死亡中に呼ぶと半復活するなど）
- トレードオフ（例: `TabListManager.tick()`がO(n²)なのは仕様であり最適化してはいけない、という注記）
- 将来消してよい一時的措置とその条件（`worldreload`/`extrareload`エイリアス、`mavenLocal()`の一時参照など）

単純なgetter/setterやフィールド定義には書かない。コメントを書くときは「これを消したら何が壊れるか」が分かる文にする。

## fail-fast と null-guard の使い分け

- **同一jar内の必須依存**（例: GuiModuleが依存するStatusModule）→ `orElseThrow(() -> new IllegalStateException(...))` で `onEnable` 時点で即死させる。登録順序が保証されている前提を信頼してよい。
- **別プラグイン・soft dependency**（`orelia-debug`から見た`WorldDebugApi`/`ExtraDebugApi`、`orelia-serverutil`から見たOreliaCoreの各Api、`DungeonModule`から見た未来ブロックの`PartyApi`など）→ 必ず`null`ガード（または`Optional`）にして、「未インストール/未初期化」を例外にせず穏当に扱う（メッセージで通知、機能を静かにスキップ、など）。

新しいコードを書くとき、依存先が「同じjarの中で自分より先に登録される」のか「そうでない可能性がある」のかを最初に確認し、それに応じてどちらのパターンにするか決める。

## テスト方針

- Bukkitの`Material`/`ItemStack`/`Player`などサーバー起動が要る型に依存するロジックは、そのままでは単体テストできない。
- **純粋なロジック部分（添字計算、数式、優先度ソートなど）を、Bukkit型に依存しないpackage-private staticメソッドとして切り出し、そこだけJUnitで直接叩く。** 実例:
  - `DamageFormula`（ダメージ計算式、Bukkit非依存で完全にpure）
  - `GuiPaginator.pageStart/pageEnd/hasPreviousPage/hasNextPage`（ページ送りの添字計算）
  - `RegionQueryService.orderByEffectivePriority`（WorldGuard非依存の優先度トポロジカルソート）
- 新しい計算ロジックを書くときは、最初からこの形（Bukkit型を引数に取らない）で書けないか検討する。書けないなら無理にテストを増やそうとしない（`orelia-serverutil`のVelocity側コードのようにstatic reviewのみで済ませている領域もある）。

## 複数リポジトリをまたぐ開発

- `orelia-debug`と`orelia-serverutil`は`orelia-core`をjitpack経由（GitHubから）で参照する。`orelia-core`側を並行して直しながら試すときは、`orelia-core`で`./gradlew publishToMavenLocal`してから、依存側の`build.gradle.kts`の`mavenLocal()`エントリ（jitpackより手前に置かれている、一時的なもの）が拾う。**この`mavenLocal()`行は指示なく消さない**（本番リリース前に消す前提のコメント付き）。
- `orelia-serverutil`はOreliaCoreが**soft dependency**（`softdepend`、`depend`ではない）なので、`rpg.core.*`を直接importしてはいけない。代わりに`rpg.serverutil.paper.config`/`.message`/`.util.ColorUtil`という**独立した軽量コピー**を持つ。同じようなクラスが複数リポジトリに重複しているのを見つけても、それは意図的な設計（hard dependency前提のコード共有はしない）なので、安易に「共通化」しようとしない。
- `orelia-debug`は逆にOreliaCoreが**hard dependency**なので、`rpg.core.config.ConfigManager`/`rpg.core.message.MessageManager`をそのまま再利用してよい。
- コミット時はREADME.md（と英語版があればREADME_EN.md）も更新する、という規約が各リポジトリのCLAUDE.mdに明記されている。コード変更だけでコミットを終わらせない。
- `orelia-docs`はプレイヤー・運営者向けドキュメントで、実装詳細（クラス名/メソッドシグネチャ/SQLスキーマ）を書く場所ではない。実装を変えたら、まずソース（コマンドクラスの`onCommand`ロジック）を読み直してから該当ページを直す（登録時の一行説明文はズレていることがあるので信用しない）。

## 関連知識

- 4リポジトリの全体像・モジュール登録順序: [`../../knowledge/architecture.md`](../../knowledge/architecture.md)
- カスタムカラーコード対応表: [`../../knowledge/color-codes.md`](../../knowledge/color-codes.md)
