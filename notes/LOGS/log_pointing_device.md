# ポインティングデバイス調査ログ

物理ポインティングデバイス増設に関する調査記録. append-only.

---

## 2026-05-12: Adafruit 512 (Mini Analog 2-Axis Thumb Joystick) 増設可否

### 要件

Adafruit 512 は X 軸・Y 軸それぞれ 10 kΩ ポテンショメータのアナログ出力 + モーメンタリ Select (SEL) ボタンを持つ. 最低必要 Input/Output (I/O) は Analog-to-Digital Converter (ADC) × 2 本(X/Y)+ デジタル入力 × 1 本(SEL).

### nice!nano v2 で外部利用可能な ADC ピン

ZMK v0.3.0 の `app/boards/arm/nice_nano/arduino_pro_micro_pins.dtsi` の `gpio-map` から導出:

| Pro Micro パッド | nRF52840 General Purpose Input/Output (GPIO) | ADC チャネル |
|---|---|---|
| D19 | P0.02 | AIN0 |
| D20 | P0.29 | AIN5 |
| D21 | P0.31 | AIN7 |

加えて P0.04 (AIN2) はバッテリー電圧監視に占有・外部アクセス不可. AIN1/3/4/6 は Pro Micro 互換のため Printed Circuit Board (PCB) に引き出されていない.

**外部利用可能な ADC は 3 本(D19/D20/D21)のみ**.

### 現在の Corne 配線(ZMK v0.3.0 標準)

- Rows (`corne.dtsi`): pro_micro 4, 5, 6, 7
- Cols (`corne_left.overlay`): pro_micro 21, 20, 19, 18, 15, 14
- Inter-Integrated Circuit (I2C) bus (nice!view): pro_micro 2 (P0.17 SDA), pro_micro 3 (P0.20 SCL)

**ADC ピン D19/D20/D21 はすべて `col-gpios` に占有されている**.

### 結論と迂回策

現状の `corne_left.overlay` を保ったまま Adafruit 512 を ADC 直結することは不可能. 迂回策を実現容易性順に整理:

| 案 | 内容 | 配線変更 | 評価 |
|---|---|---|---|
| A | ADS1115 等の I2C 接続 ADC を増設し, ジョイスティック X/Y を AIN0/AIN1 に接続. I2C は nice!view と共用 | なし | 第一候補. Zephyr 側に `adc-ads1x15` ドライバあり, ZMK 用のインプットドライバ実装は必要 |
| B | `five_column_transform` で未使用となっている列ピン(pro_micro 21 か 14)を解放し, その 1 本を ADC として使用. 残り 1 本は案 A と組み合わせ | あり | マトリクス変換と実機配線の突き合わせ調査が前提 |
| C | シールド定義を上書きして `col-gpios` を非 ADC ピン(D0/D1/D4-D10/D16)へ移し, D19/D20/D21 を解放 | あり | パッド再はんだ付け必須. ハードル高 |
| D | Adafruit 512 を諦め, I2C 完成品ジョイスティック(例: Adafruit STEMMA Mini I2C Joystick)へ置換 | なし | ZMK 既存ドライバの恩恵が大きい. 要件次第で有力 |
| E | Pulse-Width Modulation (PWM) センサ系(PMW3360 等)の光学センサへ転向 | 大 | ADC 制約から完全解放されるが構成変更大 |

### 推奨

- アナログジョイスティックを保持したい場合: **案 A** (ADS1115 を I2C 増設, SEL ボタンは未使用の D0/D1 等へ)
- デバイスを問わない場合: **案 D** (I2C ジョイスティック)

詳細な選定と実装着手は未実施. `notes/TODO.md` 参照.

---

## 2026-06-16: FJ08K-B10K 採用決定と実装方針

### 採用デバイス

FJ08K-B10K: 2 軸アナログジョイスティック, 各軸 10 kΩ ポテンショメータ, 自動センタリング, THT 5 ピン(VCC, GND, X, Y, SW), 外形寸法 約 17×17×16 mm.

Adafruit 512 は ADC ピン不足で断念. I2C 経由案(ADS1115 等)も検討したが, FJ08K を右手側ペリフェラルに実装する構成で ADC ピンを確保できることが判明し, アナログ直結に回帰.

### セントラル / ペリフェラル割当

右手側をペリフェラルのまま変更しない. ZMK の `zmk,input-split` がペリフェラル側のポインティングデバイスイベントを BLE 経由でセントラルへ転送する機能を持つため, セントラルを右に変更する必要はない.

### 右手側の ADC ピン確保策

`corne_right.overlay` の `col-gpios` 調査結果:

| Col | Pro Micro パッド | nRF52840 GPIO | ADC | 5col での使用状況 |
|---|---|---|---|---|
| 1 | D14 | P1.11 | なし | 使用中 |
| 2 | D15 | P1.13 | なし | 使用中 |
| 3 | D18 | P1.15 | なし | 使用中 |
| 4 | D19 | P0.02 | AIN0 | 使用中 |
| 5 | D20 | P0.29 | AIN5 | 使用中 |
| 6 | D21 | P0.31 | AIN7 | **物理未使用** (5col のため) |

- nice!view は SPI0 を使用: D1 (CS/P0.06), D2 (MOSI/P0.17), D3 (SCK/P0.20)
- nice!nano v2 背面の"Extra GPIO"パッドは nRF52840 QFN ボールパッドであり, 手はんだは現実的でない
- 空き非 ADC ピン: D0, D8, D9, D10, D16

**確保方針:**

- ADC ピン 1: D21/P0.31/AIN7 — Col 6 が物理未使用のため, PCB 加工なしに解放済み
- ADC ピン 2: Col 5 を D20 (P0.29/AIN5) から D16 (P0.10, 非 ADC) へ PCB トレースカット + ジャンパ線で移設し, D20/AIN5 を解放
- ソフトウェア: `corne_right.overlay` の `col-gpios` を書き換え(D20 → D16, D21 を削除)

### ソフトウェアアーキテクチャ

- ZMK モジュール: `zmk-analog-input-driver` (badjeff) を `config/west.yml` へ追加
- 共有 `.dtsi`: `zmk,input-split` を定義し, `zmk,input-listener` を disabled で宣言
- 右 (ペリフェラル) overlay: FJ08K アナログデバイスノードを定義し, input-split へ接続
- 左 (セントラル) overlay: `zmk,input-listener` を enable
- Kconfig: `CONFIG_ZMK_POINTING=y`, `CONFIG_ADC=y`, `CONFIG_ANALOG_INPUT=y`

実装作業は未着手. `notes/TODO.md` 参照.

---

## 2026-06-20: zmk-analog-input-driver (badjeff) 詳細調査

### モジュール基本情報

- リポジトリ: https://github.com/badjeff/zmk-analog-input-driver
- compatible: `"zmk,analog-input"`
- ZMK v0.3.0 と互換性あり(Zephyr 標準 input API のみ使用)
- タグ/リリースなし. コミット SHA `2684f22`(2026-04-06 時点)での固定を推奨

### west.yml 追加例

```yaml
- name: zmk-analog-input-driver
  remote: badjeff
  revision: 2684f22
```

### devicetree binding の主要プロパティ(各軸子ノード)

| プロパティ | 内容 |
|---|---|
| `io-channels` | ADC チャンネル(例: `<&adc 7>` = P0.31/AIN7) |
| `mv-mid` | 中点電圧 mV(要実測, 理論値 ~1650) |
| `mv-min-max` | 中点からの最大偏差 mV |
| `mv-deadzone` | デッドゾーン mV(デフォルト 10) |
| `evt-type` / `input-code` | 例: `INPUT_EV_REL` / `INPUT_REL_X` |
| `scale-multiplier` / `scale-divisor` | 感度調整 |

### ソフトウェアアーキテクチャの実現可能性確認

PLAN.md の"共有 dtsi + 右 overlay + 左 overlay + input-split/listener"構成は実現可能と確認.
ドライバは `input_report()` で Zephyr input イベントを発行するため, `zmk,input-split` にそのまま渡せる.

推奨追加モジュール: `zmk-input-processor-xyz`(BLE 帯域節約のための XY 圧縮). 採否は TODO 検討候補へ.

### 発見した課題

3 件の新規課題を `notes/ISSUES.md` に追加した:

1. **nRF52840 ADC oversampling バグ**: `battery_nrf_vddh.c` の `oversampling = 4` が analog-input-driver と競合し約 1 分後に ADC が stuck する. v0.3.0/main ともに未修正.
2. **消費電力**: ポーリングモードのため wireless には非推奨と README に明記. `sampling-hz` 低減で緩和可能だが根本解決ではない.
3. **mv-mid キャリブレーション**: 中点電圧は個体差あり. `CONFIG_ANALOG_INPUT_LOG_DBG_RAW=y` で実測が必要.

ディープスリープ時の GPIO 電源制御は既存 TODO 項目として管理中. ドライバ側に電源制御機能はなく, `regulator-fixed` + PM コールバック等での自前実装が必要な点を確認.

---

## 2026-06-20: devicetree 設計の具体化

ZMK v0.3.0 の input-split / input-listener binding 仕様, Corne シールドの overlay 構造, badjeff の参考実装を調査し, FJ08K 実装に必要な devicetree 記述を確定した.

### binding 仕様の確認

`zmk,input-split`:

| プロパティ | 必須 | 備考 |
|---|---|---|
| `reg` | 必須 | 識別整数 |
| `device` | ペリフェラル側のみ | 入力デバイスの phandle |
| `input-processors` | 任意 | BLE 送信前のプロセッサ |

`zmk,input-listener`:

| プロパティ | 必須 | 備考 |
|---|---|---|
| `device` | 必須 | split 構成では `&joystick_split` |
| `input-processors` | 任意 | 受信後のプロセッサ |

子ノードでレイヤーごとのオーバーライド可能 (`layers`, `process-next`, `input-processors`).

### ファイル構成の確定

```
config/
├── west.yml              # 既存 + badjeff モジュール追加
├── corne.conf            # 既存(両側共有)
├── corne_right.conf      # 新規: CONFIG_ADC=y, CONFIG_ANALOG_INPUT=y
├── corne.keymap          # 既存
├── corne.dtsi            # 新規: input-split + input-listener 定義(共有)
├── corne_left.overlay    # 新規: input-listener を enable
└── corne_right.overlay   # 新規: FJ08K 定義 + input-split 接続 + col-gpios 書き換え
```

`build.yaml` の変更は不要(ファイル名が正しければビルドシステムが自動検出する).

### overlay 適用の仕組み

- 公式シールド overlay とユーザー overlay は両方適用される(追加適用, 置換ではない)
- `&kscan0` の `col-gpios` は後勝ちで上書きされる
- conf は検索順にマッチした全ファイルが累積される

### input-listener の配置方法

- 方法 A: 共有 dtsi で disabled 宣言 → セントラル overlay で enable (ZMK 公式パターン) ← **採用**
- 方法 B: keymap に直接記述 (badjeff パターン)

方法 A を採用する. `status = "disabled"` で共有 dtsi に宣言し, `corne_left.overlay` で `status = "okay"` に上書きする.

### XYZ 圧縮 (zmk-input-processor-xyz)

- ペリフェラル側: `&zip_xyz` (X+Y → Z パッキング)
- セントラル側: `&zip_zxy` (Z → X+Y 展開)
- BLE 転送量を約 50% 削減. オプションだが推奨. 採否は TODO 検討候補として管理中.

### イベントフロー

```
FJ08K → ADC → anin0 (INPUT_REL_X/Y) → joystick_split (BLE 転送) → joystick_split@セントラル → joystick_listener → HID マウスレポート
```

### 参考実装

badjeff/zmk-config の corne36 構成で split_inputs の定義方法, input-processors の接続, conf の設定項目を確認した.

---

## 2026-06-20: Kconfig / west.yml / ビルド互換性の確定

devicetree 設計調査(同日第 2 回)に続き, 実装に必要な残り 3 項目を確定した.

### Kconfig 設定

`corne.conf`(共有): 変更不要. 既存の `CONFIG_ZMK_POINTING=y` / `CONFIG_ZMK_DISPLAY=y` / `CONFIG_ZMK_SLEEP=y` で十分.

`corne_right.conf`(新規・ペリフェラル専用):

```ini
CONFIG_ANALOG_INPUT=y
CONFIG_ANALOG_INPUT_REPORT_INTERVAL_MIN=22
```

`CONFIG_ADC` は `ANALOG_INPUT` が自動選択する. `CONFIG_INPUT` は `ZMK_POINTING` が自動選択する. input-split / input-listener / input-processor-xyz の各 Kconfig は DT ノードで自動 enable されるため明示不要.

`corne_left.conf`: 不要(input-listener 等はすべて DT auto-enable).

### west.yml モジュール追加

badjeff remote を追加し, 2 モジュールを固定 SHA で参照する:

```yaml
remotes:
  - name: badjeff
    url-base: https://github.com/badjeff

projects:
  - name: zmk-analog-input-driver
    remote: badjeff
    revision: 2684f22ee7e2168d4393f7e63676912210a796fc
  - name: zmk-input-processor-xyz
    remote: badjeff
    revision: 0f0574f6a6c5b08fa964dff7b957ce67b2e0a9cf
```

両モジュールとも `zephyr/module.yml` に `depends` 宣言なし. 追加の依存 project は不要.

### ビルド互換性

nice_view との GPIO/ADC 競合なし. nice_view が使用する SPI ピン(D1 CS/P0.06, D2 MOSI/P0.17, D3 SCK/P0.20)と FJ08K が使用する ADC ピン(D20/AIN5, D21/AIN7)は物理的に非重複.

`build.yaml` の変更不要.

### 発見した課題

2 件を `notes/ISSUES.md` に追加した:

1. **overlay 命名問題**: `config/corne_right.overlay` が in-tree shield では確実に適用されない可能性(ZMK Issue #1382). 対処案は案 A(devicetree 変更を corne.keymap に集約)と案 B(overlay ファイル分離)の 2 択. 未決定.
2. **badjeff モジュールの v0.3.0 互換性**:"Zephyr 標準 API のみ使用"と判定していたが, fork 前提で開発されているとの指摘もあり矛盾. 実ビルドでの検証が必要.

---

## 2026-06-20: ハードウェアピン配線の確認と PCB コラムトレース調査

### MCU → Corne シールドのピン対応確認

nice_nano_v2 (nRF52840) と Corne シールドのピン対応を ZMK ソース (`arduino_pro_micro_pins.dtsi`, `corne.dtsi`, `corne_right.overlay`) から確認した.

| 用途 | Pro Micro パッド | nRF52840 GPIO |
|---|---|---|
| Row 0 | D4 | P0.22 |
| Row 1 | D5 | P0.24 |
| Row 2 | D6 | P1.00 |
| Row 3 | D7 | P0.11 |
| Col 1 (inner) | D21 | P0.31 (AIN7) |
| Col 2 | D20 | P0.29 (AIN5) |
| Col 3 | D19 | P0.02 (AIN0) |
| Col 4 | D18 | P1.15 |
| Col 5 | D15 | P1.13 |
| Col 6 (outer) | D14 | P1.11 |

スキャン方式は col2row. この対応は PLAN.md の ADC ピン計画と整合している.

ピン対応の概要図を `notes/corne_pinmap.svg` に生成した(MCU ピンアウト, キーマトリクス, Pro Micro → nRF52840 GPIO 対応, col2row の説明を含む).

### Chocofi PCB のコラムトレース構造

pashutk/chocofi のスキーマティックを確認した結果, Chocofi PCB には 5 列キーボードにもかかわらず **6 本のコラムトレース(col0〜col5)が存在する**ことを確認した.

- D21 に対応する col0 トレースは PCB 上に物理的に存在する
- ただし col0 位置にはキースイッチが配置されていない
- PLAN.md の"D21 は物理未使用"は"キースイッチに接続されていない"という意味で正確

この構造から, D21 は配線済みトレースを持つが接続先キースイッチがないという状態であり, FJ08K 用 ADC として転用するための PCB 加工は不要である可能性が高い.

### D21 未接続の動作確認用 overlay

D21 がキースイッチに接続されていないことを実機で確認するため, `config/corne_right.overlay` を作成した.

変更内容:
- `col-gpios` から D21 (P0.31) を削除(6 エントリ → 5 エントリ)
- `col-offset` を 6 → 7 に変更(右手列番号のオフセット調整)

右手側にフラッシュして全キーが正常動作することを確認できれば, D21 がいずれのキースイッチにも接続されていないことの実証となる. 検証は未実施.
