# 設計方針

本リポジトリは ZMK (Zephyr-based keyboard firmware) のユーザー設定であり, Corne (chocofi) 5 列配列 + nice!nano v2 + nice!view を対象とする.

## 構成の確定事項

### ハードウェア構成

- 基板: nice!nano v2 (nRF52840)
- シールド: `corne_left` / `corne_right` (ZMK 標準) に `nice_view_adapter` + `nice_view`
- マトリクス変換: `&five_column_transform` を使用(5×3+3 配列)
- セントラル / ペリフェラル割当: `corne_left` 側がセントラル(ZMK 標準の `SHIELD_CORNE_LEFT` Kconfig による). 本リポジトリは標準のフラッシュ手順を踏襲し"左手 = セントラル"運用

### ZMK バージョン

- `config/west.yml` で `zmkfirmware/zmk@v0.3.0` に固定
- バージョン更新時はディスプレイ Kconfig 名・kscan ドライバ仕様の差分を確認すること

### 電源管理方針

- **アイドル**: ZMK デフォルト 30 秒で遷移(Kconfig 上の ON/OFF スイッチなし). FJ08K 導入後はアイドル遷移時に ADC サンプリングを停止し MCU ウェイクアップを抑制する
- **ディープスリープ**: `CONFIG_ZMK_SLEEP=y` で有効化済み. デフォルトの 15 分無操作で発動
- **FJ08K 電源**: FJ08K の各軸ポテンショメータは常時約 0.66 mA (2 × 10 kΩ @ 3.3 V) を消費する. ディープスリープ時は GPIO で VCC を切断して消費を抑える. ジョイスティック操作単独ではディープスリープから復帰できない点は許容する(任意キー押下で復帰)
- **表示ブランク**: `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE` は設定しない(`y if SSD1306` がデフォルトで, nice!view は Sharp Memory-in-Pixel LCD のため非対象). メモリ LCD はアイドル・ディープスリープいずれでも表示を保持し(双安定), 積極的に消す利点が乏しいため

### ポインティング方針

- 現状: ZMK 内蔵のマウスエミュレーション (`CONFIG_ZMK_POINTING=y`) のみ. `&mmv` の `time-to-max-speed-ms` を 450 ms に上書き, `ZMK_POINTING_DEFAULT_MOVE_VAL` を 1500, `ZMK_POINTING_DEFAULT_SCRL_VAL` を 20 に設定済み
- **採用デバイス**: FJ08K-B10K(2 軸アナログジョイスティック, 各軸 10 kΩ ポテンショメータ, 自動センタリング, THT 5 ピン)
- **実装側**: 右手側ペリフェラル. セントラルを左のまま変更しない(ZMK の `zmk,input-split` がペリフェラル側ポインティングデバイスを BLE 経由でセントラルへ転送するため)
- **ADC ピン確保策**:
  - ADC ピン 1: Col 5 (D21/P0.31/AIN7) — 5col Chocofi では物理未使用のため PCB 加工不要
  - ADC ピン 2: Col 4 を D20 (P0.29/AIN5) から D16 (P0.10, 非 ADC) へ PCB トレースカット + ジャンパ線で移設し, D20/AIN5 を解放
  - ソフトウェア: `corne_right.overlay` の `col-gpios` を書き換え (D20 → D16, D21 を削除)
- **ZMK モジュール**: `zmk-analog-input-driver` (badjeff) を `config/west.yml` へ追加
- **ソフトウェア構成**:
  - 共有 `.dtsi`: `zmk,input-split` 定義 + `zmk,input-listener` を disabled で宣言
  - 右 (ペリフェラル) overlay: FJ08K アナログデバイス定義, input-split へ接続
  - 左 (セントラル) overlay: `zmk,input-listener` を enable
  - Kconfig: `CONFIG_ZMK_POINTING=y`, `CONFIG_ADC=y`, `CONFIG_ANALOG_INPUT=y`

### キー入力タイミング方針

- `&mt` のタッピング期間を 125 ms へ短縮(デフォルト 200 ms)
- `flavor = "balanced"` を採用
- `require-prior-idle-ms` および `quick-tap-ms` は未設定(誤判定が顕在化したら追加検討)
