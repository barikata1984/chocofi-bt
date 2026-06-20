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
