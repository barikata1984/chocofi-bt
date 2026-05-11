# TODO

## 未着手

- [ ] ディープスリープ有効化後の実機検証（15 分無操作 → ディープスリープ遷移と任意キー押下による復帰）
- [ ] `five_column_transform` 適用下で、`corne_left.overlay` の `col-gpios` 配列のうち物理的に未使用となっている列ピンを特定（pro_micro 21 = P0.31/AIN7 か pro_micro 14 = P1.11 のどちらか）
- [ ] ポインティングデバイス Adafruit 512 (Mini Analog 2-Axis Thumb Joystick) 増設方針の決定（案 A: ADS1115 を I2C で増設 / 案 D: I2C 完成品ジョイスティックへ置換）
- [ ] 必要なら `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE=y` を追加して nice!view を 30 秒で消灯させるかを判断（消費電力上の優先度は低い）

## 検討候補（採否未定）

- [ ] `&mt` への `quick-tap-ms` および `require-prior-idle-ms` 追加によるホールド誤判定対策
- [ ] BT クリア用 3 キーコンボ (`COMBO_BT_CLR`) の `timeout-ms` 緩和（デフォルト 50 ms では同時押し困難な場合）

## 完了

- [x] ディープスリープ機能の有効化（`CONFIG_ZMK_SLEEP=y` を `config/corne.conf` に追加、commit `ae997c8`）
- [x] CI ビルド失敗の修正（GitHub Actions ワークフローのタグを `@main` → `@v0.3.0` に揃え、`west.yml` と整合）
