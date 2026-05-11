# 電源管理調査ログ

ZMK の電源管理関連の Kconfig とビヘイビアに関する調査記録。append-only。

---

## 2026-05-12: タイムアウト体系の整理とディープスリープ有効化

### ZMK のタイムアウト体系（4 系統）

| 系統 | 設定例 | 役割 |
|---|---|---|
| 電源管理 | `CONFIG_ZMK_IDLE_TIMEOUT`, `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT` | 無操作時間の累積で省電力状態へ遷移 |
| キー入力判定 | `tapping-term-ms`, `quick-tap-ms`, `require-prior-idle-ms` | 単一キーの押下時間／直近押下からの経過 |
| コンボ判定 | コンボの `timeout-ms`, `require-prior-idle-ms` | 複数キー同時押し成立窓 |
| ポインティング | `time-to-max-speed-ms` (`&mmv`) | マウス移動の最大速度到達時間 |

### 本リポジトリの設定状況（調査時点）

- `tapping-term-ms = 125`（`config/corne.keymap:21`、`&mt` グローバル上書き）
- `time-to-max-speed-ms = 450`（`config/corne.keymap:17`、`&mmv` グローバル上書き）
- 上記以外のタイムアウト関連 Kconfig は `config/corne.conf` に未記述

### アイドルとディープスリープの違い

- **アイドル状態**: `CONFIG_ZMK_IDLE_TIMEOUT` (デフォルト 30000 ms) で遷移。Kconfig 上の ON/OFF スイッチは存在せず常時有効。Central Processing Unit (CPU) クロック低下と Bluetooth Low Energy (BLE) 通信頻度低下によりバックグラウンドで省電力化
- **ディープスリープ**: `CONFIG_ZMK_SLEEP=y` が必須（デフォルト `n`）。有効時は `CONFIG_ZMK_IDLE_SLEEP_TIMEOUT` (デフォルト 900000 ms = 15 分) で遷移。nice!nano v2 で約 20 μA まで電流が下がる
- 復帰は `kscan` ノードの `wakeup-source` プロパティによる Global Purpose Input/Output (GPIO) 割込み。任意キー押下で復帰するが、分割キーボードではセントラル側のキーを押すのが確実

### nice!view の表示が消えない件

`CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE` の Kconfig 定義（v0.3.0 `app/src/display/Kconfig`）：

```kconfig
config ZMK_DISPLAY_BLANK_ON_IDLE
    bool "Blank display on idle"
    default y if SSD1306
```

nice!view は Sharp Memory-in-Pixel LCD (LS011B7DH03) であり SSD1306 ではないため、デフォルトでブランクされない。これは仕様通りの挙動であって不具合ではない。メモリ LCD は表示維持コストが極めて低く、積極的にブランクする利得が乏しいための判断と推察される。

### 実施した変更

`config/corne.conf` に以下を追記し、ディープスリープを有効化（commit `ae997c8`、push 済み）：

```conf
# Enable deep sleep (idle sleep timeout defaults to 15 minutes)
CONFIG_ZMK_SLEEP=y
```

これにより 15 分無操作でディープスリープへ遷移する。実機検証は未実施。
