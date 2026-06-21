# TODO

## 未着手

- [ ] PCB 再設計: EasyEDA Pro でインポートした chocofi プロジェクトの目視確認 (元 PCB との差分検証)
- [ ] PCB 再設計: easyeda-copilot エクステンションを EasyEDA Pro に導入
- [ ] PCB 再設計: easyeda-copilot MCP サーバーを Claude Code に接続
- [ ] PCB 再設計: LLM 支援で D20→D16, D19→D10 のピン変更 + アナログジャック追加を試行
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
- [x] ディープスリープ有効化後の実機検証 → 15 分無操作でディープスリープ遷移, 任意キー押下で復帰を確認. nice!view もディープスリープで消灯する. この挙動で十分
- [x] nice!view 消灯検討 → ディープスリープ時に自動消灯するため `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE` の追加は不要と判断
- [x] D21 未接続の確認 → `#ifdef RIGHT_HALF` で col-gpios から D21 削除済み (D14, D15, D18, D10, D20 の 5 ピン構成). 実機で全キー正常動作を確認
