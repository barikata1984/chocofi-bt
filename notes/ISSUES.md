# 未解決の技術課題

## [FJ08K] nRF52840 ADC oversampling バグ (重要)

ZMK の `battery_nrf_vddh.c` が `oversampling = 4` を設定しており, `zmk-analog-input-driver` と同時使用すると約 1 分後に ADC が stuck する既知問題.
nice!nano v2 は VDDH 経由バッテリー読み取りを使うため確実に影響する.
v0.3.0 でも main でも未修正(`.oversampling = 4` のまま).

対処候補:
- ZMK をフォークして `oversampling = 0` に変更(保守コスト高)
- `CONFIG_ANALOG_INPUT_USE_DTS_ADC_CH_CFG=y` で回避を試みる(効果要検証)

## [FJ08K] 消費電力: analog-input-driver はポーリングモード

`zmk-analog-input-driver` README が"wireless builds には非推奨"と明記.
ADC を常時ポーリングするため消費電力が増大する.
`sampling-hz` を下げることで緩和は可能だが根本解決ではない.

## [FJ08K] ディープスリープ時の GPIO 電源制御が未実装

ドライバに電源制御機能がない. ディープスリープ時の FJ08K VCC 切断は `regulator-fixed` + PM コールバック等で自前実装が必要.

## [FJ08K] mv-mid キャリブレーションが個体依存

FJ08K 中点電圧は個体差あり. `CONFIG_ANALOG_INPUT_LOG_DBG_RAW=y` で実測してから `mv-mid` を設定する必要がある.

## [FJ08K] overlay 命名問題 (重要)

`config/corne_right.overlay` が in-tree shield ではビルドシステムに確実に適用されない可能性がある(ZMK Issue #1382).

対処案:
- 案 A: devicetree 変更を `corne.keymap` に集約(確実に適用される)
- 案 B: overlay ファイル分離(PLAN.md の現方針だが適用されないリスクあり)

方針未決定. devicetree 実装着手前に確定が必要.

## [FJ08K] badjeff モジュールの ZMK v0.3.0 互換性が未検証

`zmk-analog-input-driver` / `zmk-input-processor-xyz` は fork 前提で開発されているとの指摘があり, upstream v0.3.0 でビルドできない可能性がある.
一方で"Zephyr 標準 API のみ使用"との調査結果もあり, 実ビルドでの確認が必要.
