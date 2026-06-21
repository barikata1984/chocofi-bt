# TODO

## 未着手

- [ ] ディープスリープ有効化後の実機検証(15 分無操作 → ディープスリープ遷移と任意キー押下による復帰)
- [ ] 必要なら `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE=y` を追加して nice!view を 30 秒で消灯させるかを判断(消費電力上の優先度は低い)
- [ ] D21 未接続の再確認: `DTS_EXTRA_CPPFLAGS=-DRIGHT_HALF` + `#ifdef RIGHT_HALF` で D21 削除 + col-offset=7 を正しく適用し右手側をフラッシュ → 全キー正常動作を確認すること(前回の検証は `#ifdef CONFIG_SHIELD_CORNE_RIGHT` が機能せず無効だった)
- [ ] FJ08K-B10K 実装: `zmk-analog-input-driver` + `zmk-input-processor-xyz` (badjeff) を `config/west.yml` へ追加(revision 確定済み)
- [ ] FJ08K-B10K 実装: right-side-only devicetree 変更を `corne.keymap` の `#ifdef RIGHT_HALF` ブロックに実装(`DTS_EXTRA_CPPFLAGS=-DRIGHT_HALF` via build.yaml cmake-args)
- [ ] FJ08K-B10K 実装: `corne_right.conf` を新規作成(`CONFIG_ANALOG_INPUT=y` + `CONFIG_ANALOG_INPUT_REPORT_INTERVAL_MIN=22`)
- [ ] FJ08K-B10K 実装: right overlay / left overlay / shared `.dtsi` の作成(`zmk,input-split` 構成); Col 4 を D10 (P0.09) へ移設済み(D19/AIN0 解放)
- [ ] FJ08K-B10K 実装: 実ビルドで badjeff モジュールと ZMK v0.3.0 の互換性を検証
- [ ] FJ08K-B10K 実装: ディープスリープ時に GPIO で VCC を切断する電源制御回路の設計
- [ ] FJ08K-B10K 実装: nRF52840 ADC oversampling バグへの対処(`CONFIG_ANALOG_INPUT_USE_DTS_ADC_CH_CFG=y` の効果検証, または ZMK フォークで `oversampling = 0` に変更)
- [ ] FJ08K-B10K 実装: `CONFIG_ANALOG_INPUT_LOG_DBG_RAW=y` で FJ08K 中点電圧 (`mv-mid`) を実測しキャリブレーション

## 検討候補(採否未定)

- [ ] `&mt` への `quick-tap-ms` および `require-prior-idle-ms` 追加によるホールド誤判定対策
- [ ] BT クリア用 3 キーコンボ (`COMBO_BT_CLR`) の `timeout-ms` 緩和(デフォルト 50 ms では同時押し困難な場合)
- [ ] `zmk-input-processor-xyz` (badjeff) の採用検討(BLE 帯域節約のための XY 圧縮. FJ08K 実装時に消費帯域を実測してから判断)

## 完了

- [x] ディープスリープ機能の有効化(`CONFIG_ZMK_SLEEP=y` を `config/corne.conf` に追加, commit `ae997c8`)
- [x] CI ビルド失敗の修正(GitHub Actions ワークフローのタグを `@main` → `@v0.3.0` に揃え, `west.yml` と整合)
- [x] `five_column_transform` 適用下での未使用列ピン特定 → Col 5 (D21/P0.31/AIN7) が物理未使用と確認
- [x] ポインティングデバイス増設方針の決定 → FJ08K-B10K (右手側ペリフェラル, `zmk,input-split` 経由) を採用
- [x] overlay 命名問題の方針決定 → `DTS_EXTRA_CPPFLAGS=-DRIGHT_HALF` を `build.yaml` の右手側 cmake-args に追加し, `corne.keymap` の `#ifdef RIGHT_HALF` ブロックで DT 変更を実装. 実機両側フラッシュ・全キー正常動作を確認
