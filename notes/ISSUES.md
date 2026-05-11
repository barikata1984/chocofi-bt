# 未解決の技術課題

## 構成上の制約

### Adafruit 512 ジョイスティックを ADC 直結で増設できない

`corne_left.overlay` の `col-gpios` 配列が、nice!nano v2 上で外部利用可能な ADC ピン（D19 = P0.02/AIN0、D20 = P0.29/AIN5、D21 = P0.31/AIN7）の 3 本すべてを占有している。Adafruit 512 は X/Y の 2 軸アナログ + Select (SEL) ボタン 1 本を要するため、現状の `corne_left.overlay` を保ったままでは追加できない。

- 影響: 物理ポインティングデバイス導入時にハード構成変更（配線・シールド定義書き換え）または I2C 経由 ADC 増設が不可避
- 暫定方針: `notes/PLAN.md` の「ポインティング方針」に従い I2C 経由を優先

### `five_column_transform` での未使用列ピン未確定

本リポジトリは `&five_column_transform` を用いるが、`corne_left.overlay` 側は 6 本の `col-gpios` をそのまま定義している。標準 `corne.dtsi` のマトリクス変換と `five_column_transform` を突き合わせ、実際に使われていない列ピン（pro_micro 21 か pro_micro 14）を特定すれば ADC ピン 1 本を解放できる可能性がある。未調査。

## バージョン依存リスク

### ZMK v0.3.0 ピン留めの陳腐化

`config/west.yml` で `zmk@v0.3.0` を固定しているため、本家側で進む API・Kconfig 改名（例: `CONFIG_ZMK_POINTING` の整備状況、`require-prior-idle-ms` の挙動更新）の恩恵を受けられない。アップデート時には `corne.keymap` の `&mt` / `&mmv` 周りと、`CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE` 等のディスプレイ系 Kconfig の差分確認が必要。

なお、`west.yml` で ZMK 本体をピン留めする際は GitHub Actions ワークフロー (`.github/workflows/build.yml`) の `uses:` も同じタグへ揃える必要がある。`@main` のままにすると、後方非互換変更（例: `west boards --format "{qualifiers}"` の追加）で `Check if building a board without explicit ZMK compat` ステップが KeyError で落ちる。
