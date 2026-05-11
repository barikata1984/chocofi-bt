# 設計方針

本リポジトリは ZMK (Zephyr-based keyboard firmware) のユーザー設定であり、Corne (chocofi) 5 列配列 + nice!nano v2 + nice!view を対象とする。

## 構成の確定事項

### ハードウェア構成

- 基板: nice!nano v2 (nRF52840)
- シールド: `corne_left` / `corne_right` (ZMK 標準) に `nice_view_adapter` + `nice_view`
- マトリクス変換: `&five_column_transform` を使用（5×3+3 配列）
- セントラル / ペリフェラル割当: `corne_left` 側がセントラル（ZMK 標準の `SHIELD_CORNE_LEFT` Kconfig による）。本リポジトリは標準のフラッシュ手順を踏襲し「左手 = セントラル」運用

### ZMK バージョン

- `config/west.yml` で `zmkfirmware/zmk@v0.3.0` に固定
- バージョン更新時はディスプレイ Kconfig 名・kscan ドライバ仕様の差分を確認すること

### 電源管理方針

- **アイドル**: ZMK デフォルト 30 秒で遷移（Kconfig 上の ON/OFF スイッチなし）
- **ディープスリープ**: `CONFIG_ZMK_SLEEP=y` で有効化済み。デフォルトの 15 分無操作で発動
- **表示ブランク**: `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE` は設定しない（`y if SSD1306` がデフォルトで、nice!view は Sharp Memory-in-Pixel LCD のため非対象）。メモリ LCD は表示維持コストが極めて低く積極的に消す利点が乏しいため

### ポインティング方針

- 現状: ZMK 内蔵のマウスエミュレーション (`CONFIG_ZMK_POINTING=y`) のみ。`&mmv` の `time-to-max-speed-ms` を 450 ms に上書き、`ZMK_POINTING_DEFAULT_MOVE_VAL` を 1500、`ZMK_POINTING_DEFAULT_SCRL_VAL` を 20 に設定済み
- 物理ポインティングデバイス増設時の方針:
  - 第一候補: I2C 経由 (ADS1115 増設、または I2C 完成品ジョイスティック)
  - ネイティブ Analog-to-Digital Converter (ADC) ピン直結は **非推奨**（外部利用可能な 3 本の ADC ピン D19/D20/D21 がキーマトリクス列に占有されているため、シールド定義の上書きと配線変更が必要）

### キー入力タイミング方針

- `&mt` のタッピング期間を 125 ms へ短縮（デフォルト 200 ms）
- `flavor = "balanced"` を採用
- `require-prior-idle-ms` および `quick-tap-ms` は未設定（誤判定が顕在化したら追加検討）
