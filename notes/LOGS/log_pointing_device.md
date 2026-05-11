# ポインティングデバイス調査ログ

物理ポインティングデバイス増設に関する調査記録。append-only。

---

## 2026-05-12: Adafruit 512 (Mini Analog 2-Axis Thumb Joystick) 増設可否

### 要件

Adafruit 512 は X 軸・Y 軸それぞれ 10 kΩ ポテンショメータのアナログ出力 + モーメンタリ Select (SEL) ボタンを持つ。最低必要 Input/Output (I/O) は Analog-to-Digital Converter (ADC) × 2 本（X/Y）+ デジタル入力 × 1 本（SEL）。

### nice!nano v2 で外部利用可能な ADC ピン

ZMK v0.3.0 の `app/boards/arm/nice_nano/arduino_pro_micro_pins.dtsi` の `gpio-map` から導出：

| Pro Micro パッド | nRF52840 General Purpose Input/Output (GPIO) | ADC チャネル |
|---|---|---|
| D19 | P0.02 | AIN0 |
| D20 | P0.29 | AIN5 |
| D21 | P0.31 | AIN7 |

加えて P0.04 (AIN2) はバッテリー電圧監視に占有・外部アクセス不可。AIN1/3/4/6 は Pro Micro 互換のため Printed Circuit Board (PCB) に引き出されていない。

**外部利用可能な ADC は 3 本（D19/D20/D21）のみ**。

### 現在の Corne 配線（ZMK v0.3.0 標準）

- Rows (`corne.dtsi`): pro_micro 4, 5, 6, 7
- Cols (`corne_left.overlay`): pro_micro 21, 20, 19, 18, 15, 14
- Inter-Integrated Circuit (I2C) bus (nice!view): pro_micro 2 (P0.17 SDA), pro_micro 3 (P0.20 SCL)

**ADC ピン D19/D20/D21 はすべて `col-gpios` に占有されている**。

### 結論と迂回策

現状の `corne_left.overlay` を保ったまま Adafruit 512 を ADC 直結することは不可能。迂回策を実現容易性順に整理：

| 案 | 内容 | 配線変更 | 評価 |
|---|---|---|---|
| A | ADS1115 等の I2C 接続 ADC を増設し、ジョイスティック X/Y を AIN0/AIN1 に接続。I2C は nice!view と共用 | なし | 第一候補。Zephyr 側に `adc-ads1x15` ドライバあり、ZMK 用のインプットドライバ実装は必要 |
| B | `five_column_transform` で未使用となっている列ピン（pro_micro 21 か 14）を解放し、その 1 本を ADC として使用。残り 1 本は案 A と組み合わせ | あり | マトリクス変換と実機配線の突き合わせ調査が前提 |
| C | シールド定義を上書きして `col-gpios` を非 ADC ピン（D0/D1/D4-D10/D16）へ移し、D19/D20/D21 を解放 | あり | パッド再はんだ付け必須。ハードル高 |
| D | Adafruit 512 を諦め、I2C 完成品ジョイスティック（例: Adafruit STEMMA Mini I2C Joystick）へ置換 | なし | ZMK 既存ドライバの恩恵が大きい。要件次第で有力 |
| E | Pulse-Width Modulation (PWM) センサ系（PMW3360 等）の光学センサへ転向 | 大 | ADC 制約から完全解放されるが構成変更大 |

### 推奨

- アナログジョイスティックを保持したい場合: **案 A** （ADS1115 を I2C 増設、SEL ボタンは未使用の D0/D1 等へ）
- デバイスを問わない場合: **案 D** （I2C ジョイスティック）

詳細な選定と実装着手は未実施。`notes/TODO.md` 参照。
