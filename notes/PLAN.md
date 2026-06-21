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
  - ADC ピン 1: Col 5 (D21/P0.31/AIN7) — 5col Chocofi では物理未使用のため PCB 加工不要と推定(ファームウェアによる実証は未完了)
  - ADC ピン 2: Col 4 を D19 (P0.02/AIN0) から D10 (P0.09, 非 ADC) へ PCB トレースカット + ジャンパ線で移設済み. D19/AIN0 を解放 (当初 D20→D16 の計画だったが, 実際に切断したのは D19 トレース)
  - ソフトウェア: `corne.keymap` の `#ifdef RIGHT_HALF` ブロックで `col-gpios` を書き換え (D19 → D10, D21 を削除); `build.yaml` の右手側 cmake-args に `DTS_EXTRA_CPPFLAGS=-DRIGHT_HALF` を設定
- **ZMK モジュール**: `zmk-analog-input-driver` (badjeff) を `config/west.yml` へ追加
- **per-side DTS 条件分岐方法 (解決済み)**: `#ifdef CONFIG_SHIELD_CORNE_RIGHT` は DTS プリプロセス時に展開されない(Kconfig シンボルは DTS には渡らない)ため, これを使った col-gpios 変更は一度も適用されていなかった. 正しい方法は `build.yaml` の右手側 cmake-args に `DTS_EXTRA_CPPFLAGS=-DRIGHT_HALF` を追加し, `corne.keymap` で `#ifdef RIGHT_HALF` を使うこと. 実機両側フラッシュ・全キー正常動作で確認済み. `corne_right.overlay` は使用しない.
- **右手側 col-gpios の順序**: 標準 `corne_right.overlay` は D14, D15, D18, D19, D20, D21 の順(左手側は D21 first の逆順). 右手側 col-gpios を変更する際はこの順序を維持すること.
- **ソフトウェア構成**:
  - `config/corne.dtsi` (新規, 共有): `zmk,input-split` 定義 + `zmk,input-listener` を `status = "disabled"` で宣言
  - `config/corne.keymap` (既存): 右手側専用 devicetree 変更 (`col-gpios` 書き換え D19→D10, D21 削除, col-offset 調整) を `#ifdef RIGHT_HALF` ブロックに記述. keymap はすべての overlay 処理後に適用されるため shield ラベルが利用可能
  - `config/corne_right.overlay`: **使用しない**. 右手側 devicetree 変更は `corne.keymap` の `#ifdef RIGHT_HALF` ブロックで行う
  - `config/corne_left.overlay`: **使用しない**. `zmk,input-listener` の enable は別の方法で行う(要検討)
  - `config/corne_right.conf` (新規): `CONFIG_ANALOG_INPUT=y` + `CONFIG_ANALOG_INPUT_REPORT_INTERVAL_MIN=22` のみ記載. `CONFIG_ADC` は `ANALOG_INPUT` が自動選択, `CONFIG_INPUT` は `ZMK_POINTING` が自動選択するため明示しない. input-split / input-listener / input-processor-xyz も DT ノードで自動 enable
  - `config/corne_left.conf`: 不要(input-listener 等はすべて DT auto-enable)
  - `config/corne.conf` (既存): 変更不要. `CONFIG_ZMK_POINTING=y` / `CONFIG_ZMK_DISPLAY=y` / `CONFIG_ZMK_SLEEP=y` が既存のまま有効
  - input-listener の配置方法: 共有 dtsi で disabled 宣言 → セントラル overlay で enable する ZMK 公式パターンを採用(overlay は使わず `#ifdef` guard で代替する可能性あり)
  - `build.yaml`: 右手側ビルドの cmake-args に `DTS_EXTRA_CPPFLAGS=-DRIGHT_HALF` を追加済み. 左手側には追加しない
- **west.yml モジュール追加** (revision 確定済み):
  - badjeff remote: `url-base: https://github.com/badjeff`
  - `zmk-analog-input-driver`: revision `2684f22ee7e2168d4393f7e63676912210a796fc`
  - `zmk-input-processor-xyz`: revision `0f0574f6a6c5b08fa964dff7b957ce67b2e0a9cf`
  - 両モジュールとも追加の依存 project は不要

### キー入力タイミング方針

- `&mt` のタッピング期間を 125 ms へ短縮(デフォルト 200 ms)
- `flavor = "balanced"` を採用
- `require-prior-idle-ms` および `quick-tap-ms` は未設定(誤判定が顕在化したら追加検討)
