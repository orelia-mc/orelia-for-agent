---
name: orelia-gui-screen
description: orelia-core（Minecraft Paperプラグイン）でインベントリGUI画面を新規作成・変更するときに使う。ItemBuilder/ColorUtil/Gui/GuiButton/GuiListenerの使い方、&%カスタムカラーコード、HP/EXPバーの&m表記、クリック時のアイテム名・lore更新パターン、確認画面、ページネーションなど、rpg.guiパッケージに実在する習性をまとめている。
---

# Orelia GUI画面の作り方

## 前提: GUI処理は `rpg.gui.framework` に集約する

これはコメントに明記された明示的なコーディングルール（SOW: 仕様書由来のルール）:

> GUI処理はGUIパッケージへ集約すること

- 各画面クラス（`rpg.gui.screen.*`、あるいは `rpg.<feature>.gui.*`）は Bukkit の `Inventory` を直接いじらない。必ず `rpg.gui.framework.Gui`/`GuiButton` を組み立てて返す。
- クリック/ドラッグのイベント処理は `rpg.gui.framework.GuiListener` 一箇所に集約されている。画面クラスやサービスクラスに `@EventHandler onInventoryClick` を新設しない。
- 画面固有の特殊なクリック挙動（後述の装備スロットなど）が必要な場合だけ、専用の `Listener` を追加して `GuiHolder` 経由で対象の `Gui` を判別する（`StatusEquipmentSlotListener` が実例）。

新しい画面を追加するときは、まず既存の `rpg.gui.screen.*` や `rpg.<feature>.gui.*` の中から一番近い形のクラスを探し、その構造を真似ること。

## 画面クラスの基本形

```java
public final class XxxGuiScreen {
    public static final String TAG = "xxx"; // 状態が変わる画面だけ必要（後述）

    public Gui build(Player player) {
        Gui gui = new Gui(guiConfig.title("xxx", "&%8デフォルトタイトル"), 27);
        // ... gui.set(slot, button) ...
        return gui;
    }
}
```

- タイトルは `GuiConfig#title(key, defaultTitle)` 経由にして `gui.yml: titles.<key>` でサーバー運営者が上書きできるようにする（コード直書きのデフォルト文字列は必ず用意する）。
- 画面クラスは状態を持たない one-shot ビルダー。`build(...)` を呼ぶたびに新しい `Gui`/`ItemStack` 一式を作る。`onConfirm`/`onCancel` のような処理は呼び出し側が捕まえたクロージャとして渡す（`ConfirmGuiScreen` を参照）。

## アイテムは `ItemBuilder` + `&%` カスタムカラーコードで作る

```java
new ItemBuilder(Material.NETHER_STAR)
        .name("&%eレリック厳選")
        .lore(List.of("&%7費用: &%f" + (long) cost))
        .build();
```

- `ItemBuilder`（`rpg.util.ItemBuilder`）が name/lore/customModelData/unbreakable/hiddenEnchant/PDCタグ付けをまとめて面倒を見る。生の `ItemStack`+`ItemMeta` 操作を書き散らさない。
- 色コードは3系統ある（`rpg.util.ColorUtil`）:
  - vanilla legacy: `&a`, `&c` など
  - hex: `&#RRGGBB`
  - **カスタム: `&%<char>`** — vanillaの色と被らない専用パレット（`ColorUtil.CUSTOM_COLORS`に定義済み、`0-9`+`a-h`）
- **GUIのラベル/loreは基本的に `&%` カスタムコードで書く**のが実際の習性（`&%8`=ダークグレーのタイトル、`&%e`=見出し黄、`&%7`=説明グレー、`&%f`=強調白、`&%c`=警告赤 など）。新しい画面を書くときも既存の色の使い分けに揃えること。
- `SkullMeta` など `ItemBuilder` がフックを持たない特殊メタ（プレイヤーヘッドの `setOwningPlayer` など）は `ItemBuilder` を使わず生で組み立ててよい（`StatusGuiScreen#headIcon` が実例）。

## バー表現は `&m`（打ち消し線）+スペースの繰り返し

HP/EXP/クエスト進捗などの「バー」は、専用のテクスチャ画像を使わず、**打ち消し線コード `&m` でスペースを連続させて線を引く**手法で実装されている:

```java
String bar = filledColor + "&m" + " ".repeat(filled) + "&r"
           + emptyColor  + "&m" + " ".repeat(empty)  + "&r";
```

実例: `QuestObjectiveBarRenderer`, `ActionBarService`（EXPバー）, `MonsterHealthBarRenderer`。

新しくバーを追加するときはこのパターンをそのまま使う。ポイント:
- 塗りつぶし部分と空白部分で色を変え、それぞれ `&m` を単独で入れ直す（`&m` は直前の色コードとセットで有効になるため、色を変えたら `&m` も引き直す）。
- 最後に `&r` でリセットし、後続のテキストに打ち消し線が漏れないようにする。
- 見た目の「バーの長さ」は `filled`/`empty` の文字数（スペースの個数）で調整する。

## クリックしたらアイテムの名前・loreを更新する

状態がクリックで変わる画面の更新方法は、**「画面を閉じて結果をメッセージで返す」か「同じ画面のまま該当スロットだけ差し替える」か**で作り分ける。両方の実例がある:

### パターンA: 選択して閉じる（不可逆・確認が要る操作）

`RelicUpgradeGuiScreen#handleChoice` の形:

```java
gui.set(slot, new GuiButton(icon, (player, clickType) -> handleChoice(...)));

private void handleChoice(...) {
    player.closeInventory();
    // サービス呼び出し → 失敗ならmessages.ymlキーで通知して return
    // 成功したら messages.send(...) で完了通知
}
```

破壊的操作・課金操作は同一画面のダブルクリックにせず、`ConfirmGuiScreen.build(title, description, onConfirm, onCancel)` を挟んで確認ステップを1枚挟む。

### パターンB: 画面を開いたまま該当スロットだけ書き換える（装備・在庫系）

`StatusEquipmentSlotListener` の形（`StatusGuiScreen` の装備スロット）:

1. 対象タグの `Gui` かどうかを `GuiHolder`＋`getTag()` で判定してから処理する。
2. Bukkitのカーソル入れ替えは自前でキャンセルし（`event.setCancelled(true)`）、サービス層の状態を先に更新してから、
3. **1tick遅延させて** `inventory.setItem(slot, newIcon)` と `player.setItemOnCursor(...)` を呼ぶ。

   > キャンセルされたクリックはクライアント側で表示が一旦ロールバックされるため、同tickで書き換えるとロールバックに上書きされて消える。`schedulerService.runLater(..., 1L)` で1tick遅らせるのが正解。

- 新しいアイコンは画面クラス側に `public static ItemStack xxxIcon(...)` として公開し、`build()` と専用リスナーの両方が**同じロジックで同じ見た目を作る**ようにする（`StatusGuiScreen.equipSlotIcon` が実例）。ビルド時と更新時でロジックが分岐すると見た目がズレる。

### パターンC: 時間経過で変わる値は定期リフレッシュ

プレイヤーレベルアップ・バフ増減など「どのイベントで変わるか特定しづらい」状態は、都度フックするのではなく `GuiModule` 側の `SchedulerService#runTimer` で定期的に `refresh(player, inventory)` を呼ぶ:

```java
plugin.getSchedulerService().runTimer(() ->
    plugin.getServer().getOnlinePlayers().forEach(player -> {
        var top = player.getOpenInventory().getTopInventory();
        if (top.getHolder() instanceof GuiHolder holder && StatusGuiScreen.TAG.equals(holder.getGui().getTag())) {
            statusGuiScreen.refresh(player, top);
        }
    }),
    period, period);
```

- 画面クラスは `build()` とは別に `refresh(Player, Inventory)` を持ち、`inventory.setItem(slot, icon)` で必要なスロットだけ描き直す（`Gui` を作り直さない＝開いたまま）。
- **クリックで変わるスロット（パターンB）は `refresh()` の対象から意図的に除外する。** 定期リフレッシュと専用リスナーの書き込みが競合すると、クリック直後の見た目が一瞬で上書きされる不具合になる（`StatusGuiScreen#refresh` のJavadocに明記されている設計判断）。

## ボタンとクリックルーティング

- `GuiButton(icon, action, sound)`: クリック時に `action.onClick(player, clickType)` を実行し、`sound` があれば鳴らす。デフォルト音は `GuiButton.DEFAULT_CLICK_SOUND`（`BLOCK_BAMBOO_WOOD_BUTTON_CLICK_ON`）。
- `GuiButton.display(icon)`: 非対話（見た目だけ）のアイコン。フィラーや情報表示アイコンに使う。
- `GuiListener` はボタンのある画面（`allowItemMovement()` を呼んでいない画面）では**トップインベントリ全体をデフォルトでロック**し、下段（プレイヤー自身のインベントリ）はshift-click以外自由に使わせる。
  - 例外的にアイテムの出し入れを許したいスロット（装備スロットなど）は `Gui#interactiveSlot(slot)` で個別に開放する。
  - 倉庫のような「ストレージそのもの」の画面は `Gui#allowItemMovement()` を呼んで通常のインベントリと同じ挙動にする。

## ページネーション

一覧が1画面に収まらない場合は `GuiPaginator.placePage(guiManager, gui, layout, items, page, toButton, pageBuilder)` を使う。`GuiPageLayout(itemSlots, prevSlot, nextSlot)` で画面ごとのスロット配置を渡す。前/次ページボタンは「実際にそのページが存在するときだけ」出る。新しい一覧画面を作るときは自前でページ計算をしない。

## テスト

Bukkitの `Material`/`ItemStack` に依存する部分はサーバー起動が要るため直接テストしない。ページ送りの添字計算のような **pureなロジックだけを package-private static メソッドに切り出して** JUnitで直接叩く（`GuiPaginatorTest`、`GuiPaginator.pageStart`/`pageEnd`/`hasPreviousPage`/`hasNextPage` が実例）。

## 関連知識

- カラーコード一覧: [`../../knowledge/color-codes.md`](../../knowledge/color-codes.md)
- モジュール全体の構造（GUIモジュールが他モジュールとどう繋がるか）: [`orelia-module` スキル](../orelia-module/SKILL.md)
