# orelia-for-agents

Orelia Minecraftプラグイン群（`orelia-core` / `orelia-debug` / `orelia-serverutil` / `orelia-docs`）の開発をAIエージェント（Claude Codeなど）が支援するための、**スキル集**と**参考知識ベース**を置くリポジトリ。

内容はすべて、各リポジトリの実コードと `CLAUDE.md` を読んで抽出した「実在する習性」に基づく。抽象的なベストプラクティスではなく、Oreliaのコードベースに既に存在するパターンを言語化したもの。

## 構成

```
orelia-for-agent/
├── skills/                     # タスク別のSKILL.md（Claude Codeのスキル形式）
│   ├── orelia-gui-screen/      # インベントリGUI画面の新規作成・変更
│   ├── orelia-module/          # 新しいRpgModuleの追加・変更
│   ├── orelia-combat/          # ダメージ計算・HP同期・武器強化まわりの変更
│   └── orelia-conventions/     # 横断的な細かいコーディング規約
└── knowledge/                  # スキルから参照する詳細資料
    ├── architecture.md         # 4リポジトリの関係・モジュール登録順序
    └── color-codes.md          # &%カスタムカラーコード対応表
```

## スキル一覧

| スキル | 使うタイミング |
|---|---|
| [`orelia-gui-screen`](skills/orelia-gui-screen/SKILL.md) | GUI画面（インベントリ画面）を作る・直すとき。`ItemBuilder`/`ColorUtil`/`Gui`/`GuiButton`の使い方、`&m`によるバー表現、クリック時のアイテム名・lore更新パターン、確認画面、ページネーションなど |
| [`orelia-module`](skills/orelia-module/SKILL.md) | 新機能をモジュールとして追加する・既存モジュールを直すとき。登録順序、層構造（repository/model/service/listener/command/gui）、config・messages.yml（`config-version`バンプ含む）、PlayerDataComponent、DBのSchemaOwner、コマンド登録、APIの公開範囲など |
| [`orelia-combat`](skills/orelia-combat/SKILL.md) | ダメージ計算・HP（スケール値とvanilla健康値の同期）・武器の強化値/レベルを追加・変更するとき。`DamageFormula`の固定順パイプライン、`CombatDamageListener`一本化ルール、`ScaledHealthService`同期漏れの罠、`WeaponIdentityService`の合成順など |
| [`orelia-conventions`](skills/orelia-conventions/SKILL.md) | 上記のどれにも当てはまらない、細かいスタイルや判断基準を確認したいとき。カラーコードの使い分け、Javadocの書き方、fail-fast/null-guardの使い分け、テスト方針、複数リポジトリ間の連携ルール |

各SKILL.mdは概要と手順に絞り、細かい対応表は `knowledge/` を参照する形にしてある。

## 使い方（Claude Code）

このリポジトリ単体では自動ロードされない。Claude Codeにスキルとして認識させるには、対象プロジェクト（例: `orelia-core`）側の `.claude/skills/` に、`skills/` 配下の各ディレクトリをコピーまたはシンボリックリンクする。

```bash
# 例: orelia-core側にシンボリックリンクで反映する場合
ln -s ../../orelia-for-agent/skills/orelia-gui-screen ../orelia-core/.claude/skills/orelia-gui-screen
ln -s ../../orelia-for-agent/skills/orelia-module      ../orelia-core/.claude/skills/orelia-module
ln -s ../../orelia-for-agent/skills/orelia-combat      ../orelia-core/.claude/skills/orelia-combat
ln -s ../../orelia-for-agent/skills/orelia-conventions ../orelia-core/.claude/skills/orelia-conventions
```

`knowledge/` 配下は各SKILL.mdから相対パスでリンクしているだけなので、`skills/`と`knowledge/`は常にセットで置くこと（`skills/`だけをコピーすると相対リンクが壊れる）。

## 対象範囲

内容は `orelia-core` を中心にしている（実装量・習性の密度が最も高いため）。`orelia-debug`/`orelia-serverutil`との連携やそれぞれの制約（hard/soft dependency、独自ColorUtilコピーなど）は `orelia-conventions` と `knowledge/architecture.md` に補足として含めているが、`orelia-debug`/`orelia-serverutil`固有の詳しい規約が必要な場合は、それぞれのリポジトリの `CLAUDE.md` を直接参照すること。`orelia-docs`はコードを持たないため対象外。

## メンテナンス方針

Oreliaのコード側の設計・規約が変わったら、対応する `SKILL.md`/`knowledge/*.md` も合わせて更新する。逆に、ここに書いた内容と実コードが食い違っているのを見つけたら、実コードを正としてこちらを直す（各リポジトリの `CLAUDE.md` と同じ原則）。
