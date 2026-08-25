---
name: orelia-combat
description: orelia-coreでダメージ計算・HP（スケール値とvanilla健康値の同期）・武器の強化値/レベルなど、戦闘周りのロジックを追加・変更するときに使う。DamageFormulaの固定順パイプライン、CombatDamageListenerを分岐せず一本化するルール、ScaledHealthServiceでvanilla healthを同期させる箇所、WeaponIdentityServiceの強化値×武器レベル合成をまとめている。
---

# Orelia 戦闘・HP同期の作り方

## 前提: ダメージ計算は `DamageFormula` に一本化する

`rpg.status.combat.DamageFormula`（Bukkit非依存・pure・JUnitテスト済み）が**唯一の**ダメージ計算パイプライン:

```
base attack power → ATK%（applyAttackBonus） → DEF（mitigate） → crit roll/multiplier（rollCrit/criticalMultiplier）
→ elemental weakness（applyElementalWeakness） → elemental damage bonus（applyElementalDamageBonus）
```

- 武器/素手/モンスター/スキル、どの攻撃も最終的にこの`compute(...)`を1回通す。この順序は固定で、途中の値を他の場所で計算し直さない。
- 新しいダメージ源（新スキル、新モンスターアビリティなど）を足すときも、**このメソッドの呼び出し方を変える**か引数を増やすかで対応し、`DamageFormula`の外に同じ計算を再実装しない。

## 前提: ダメージイベントのリスナーは増やさない

`rpg.monster.listener.CombatDamageListener`（`EntityDamageByEntityEvent`, `EventPriority.LOW`）が**全ての**近接/モンスターダメージイベントの唯一のリスナー。かつて`WeaponUseListener`/`CombatStatusListener`/`MonsterCombatListener`の3つに分かれていて、**同一優先度リスナー間の実行順序はBukkitでは未定義**なため、crit適用がATK%/DEFより前に来てしまうバグを踏んだ経緯がある。

- 新しい戦闘関連の仕様を足すときも、**同じイベントに新しいリスナーを追加しない**。既存の`CombatDamageListener#resolveAttack`の分岐（プレイヤー/モンスター/プロジェクタイル/スキル override/ability override）に条件を足す形にする。
- スキル・モンスターアビリティのように「攻撃力は事前に計算済みで、DEF/crit/weaknessだけこのリスナーに解決してほしい」場合は、`DamageFormula.SKILL_OVERRIDE_METADATA`／`ABILITY_OVERRIDE_METADATA`を攻撃者にスタンプする方式に乗る（`SkillDamage`、`MonsterAbilityCastService`/`BossAbilityCastService`が実例）。イベント自体をキャンセルして自前で`damage()`を呼ぶような迂回はしない。
- 完全にこのリスナーの対象外にしたい特殊なビジュアル演出（`MagicWandAbilityListener`の召喚エフェクトなど）は専用のmetadataキーで明示的に除外する（`WAND_FANGS_METADATA_KEY`が実例）。`event.setCancelled(true)`だけでは足りない——このリスナーは`ignoreCancelled`を付けていないため、キャンセルしても以降の副作用（HP減算など）は走ってしまう。

## HPを変更する処理は必ず vanilla health を同期させる

プレイヤー/タグ付きモンスターの「本当のHP」は`StatType.HP`（数百〜数千）で管理され、vanillaの体力（20ハート、モンスターは`config.yml: combat.scaled-health.vanilla-cap`で頭打ち）とは別物。両者は`rpg.status.service.ScaledHealthService`（pure・Bukkit-entity限定のユーティリティ）を通じてのみ橋渡しする:

```java
ScaledHealthService.syncVanillaHealth(entity, scaledCurrent, scaledMax);          // scaled値→vanilla値へ反映
ScaledHealthService.convertDamageToVanilla(entity, scaledDamage, scaledMax);      // scaledダメージ→vanilla換算値
```

- **戦闘ダメージ**（`EntityDamageByEntityEvent`経由）: `convertDamageToVanilla`で換算した値を`event.setDamage(...)`に渡し、Bukkit自身のイベント解決（ノックバック/被ダメ音/死亡処理）に任せる。`setHealth`を直接呼ばない。scaled側のHP減算は`StatusService#applyScaledCombatDamage`のような別呼び出しで行う。
- **それ以外の全て**（食事回復・ポーション・レベルアップ回復・環境ダメージ・join/respawn）は`currentHp`を変更した直後に`syncVanillaHealth`を呼んで揃える。環境ダメージ（`EntityDamageEvent`で`EntityDamageByEntityEvent`ではないもの＝転落・炎上・溺水など）は`CombatDamageListener`を一切通らないので、専用リスナー（`ScaledHealthEnvironmentalDamageListener`）が別途必要になる——**新しいダメージ源を足すとき「これは`EntityDamageByEntityEvent`か、ただの`EntityDamageEvent`か」を必ず確認する**。
- `syncVanillaHealth`は**死亡中のエンティティに対して何もしない**（no-op）。死亡直後〜リスポーン前に`setHealth`で0超の値を書き込むと、クライアントは死亡画面のままなのにサーバー側だけ復活してモブに狙われる不具合を実際に踏んだ経緯がある。定期処理（`tickRegen`など）でオンラインプレイヤーを巡回するときはこの前提を壊さない。
- 新しいHP変更経路（新しいバフ/デバフ、新しい回復アイテムなど）を足すときは、「`currentHp`を書き換えたら必ず同じ呼び出しの中で`syncVanillaHealth`（または戦闘イベント側なら`convertDamageToVanilla`）を呼ぶ」をチェックリストにする。片方だけ更新すると、次の同期タイミングでvanilla表示が一瞬で巻き戻る/吹き飛ぶ見た目になる。

## 武器の強化値とレベルは別物、合成順は固定

`rpg.item.service.WeaponIdentityService`が武器インスタンスのPDCカウンタを2つ独立に扱う:

- **強化値**（`enhancementLevel`/`enhance()`、上限なし）— 強化屋NPCの強化、レベルごとに攻撃力+10%
- **武器レベル**（`weaponLevel`/`levelUp()`）— `items.yml`の`level:`から開始し、プレイヤーの character level（`WeaponLevelConfig#weaponLevelCap`）でゲートされながら上げる。レベルごとに`attack-power-factor`（デフォルト5%）加算

最終攻撃力は`baseAttackPower(stack, data)`が唯一の合成箇所:

```java
attack-power * (1 + weaponLevel * weaponLevelFactor) * enhancementMultiplier
```

`CombatDamageListener`・`SkillDamage`はどちらも`WeaponData.getAttackPower()`を直接読まず、必ずこの`baseAttackPower(...)`経由にする。新しい攻撃力ボーナス要素（新しいエンチャント的な仕組みなど）を足すときも、この式を書き換えるか掛け算の因子を1つ増やす形にし、呼び出し側で個別に計算しない。

## 関連知識

- モジュール全体の構造・登録順序: [`orelia-module` スキル](../orelia-module/SKILL.md)
- 横断的な細かい規約（fail-fast/null-guard、テスト方針）: [`orelia-conventions` スキル](../orelia-conventions/SKILL.md)
