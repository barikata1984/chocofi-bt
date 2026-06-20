# TODO

## 未着手

- [ ] ディープスリープ有効化後の実機検証(15 分無操作 → ディープスリープ遷移と任意キー押下による復帰)
- [ ] 必要なら `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE=y` を追加して nice!view を 30 秒で消灯させるかを判断(消費電力上の優先度は低い)
- [ ] FJ08K-B10K 実装: PCB トレースカット + ジャンパ線(Col 4: D20 → D16)の実機加工
- [ ] FJ08K-B10K 実装: `zmk-analog-input-driver` (badjeff) を `config/west.yml` へ追加
- [ ] FJ08K-B10K 実装: right overlay / left overlay / shared `.dtsi` の作成(`zmk,input-split` 構成)
- [ ] FJ08K-B10K 実装: Kconfig 追加(`CONFIG_ZMK_POINTING=y`, `CONFIG_ADC=y`, `CONFIG_ANALOG_INPUT=y`)
- [ ] FJ08K-B10K 実装: ディープスリープ時に GPIO で VCC を切断する電源制御回路の設計

## 検討候補(採否未定)

- [ ] `&mt` への `quick-tap-ms` および `require-prior-idle-ms` 追加によるホールド誤判定対策
- [ ] BT クリア用 3 キーコンボ (`COMBO_BT_CLR`) の `timeout-ms` 緩和(デフォルト 50 ms では同時押し困難な場合)

## 完了

- [x] ディープスリープ機能の有効化(`CONFIG_ZMK_SLEEP=y` を `config/corne.conf` に追加, commit `ae997c8`)
- [x] CI ビルド失敗の修正(GitHub Actions ワークフローのタグを `@main` → `@v0.3.0` に揃え, `west.yml` と整合)
- [x] `five_column_transform` 適用下での未使用列ピン特定 → Col 5 (D21/P0.31/AIN7) が物理未使用と確認
- [x] ポインティングデバイス増設方針の決定 → FJ08K-B10K (右手側ペリフェラル, `zmk,input-split` 経由) を採用
