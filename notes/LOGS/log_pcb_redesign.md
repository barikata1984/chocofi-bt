# PCB 再設計: EasyEDA 移行と LLM 支援ワークフロー

## 2026-06-21: 初期調査

### 発端

@gclue_akira のツイート群 (2026-06-18〜19) を調査.
Codex 5.5 / Claude Opus 4.8 + EasyEDA MCP (easyeda-copilot) で Arduino 互換基板の回路設計から JLCPCB 発注画面到達まで AI エージェントが自律的に完遂した実例.
人間の介入は基板色の選択と支払いのみ.

### 目的

chocofi の PCB を改造し, 将来的に JLCPCB への基板発注まで LLM ベースのワークフローで実現したい.

具体的な変更内容:
- D20 と D19 を開けてそれぞれ D16 と D10 に機能を移動 (ADC ピン確保)
- D20 と D19 をアナログ入力用のピンとしてジャックから信号を取る構成に変更
- PCB パターンの引き直し

### ツイートのシステム構成

2 層構成:
1. **EasyEDA MCP サーバー (easyeda-copilot)**: EasyEDA Pro 内エクステンションとして動作. WebSocket (`ws://127.0.0.1:8787`) 接続. 回路図生成, LCSC 部品検索, 配線等を API 経由で操作
2. **Computer Use (Codex アプリ固有)**: MCP に露出されていない GUI メニュー (Auto Route 等) をスクリーンショット認識 + クリック操作で直接叩く

### EasyEDA vs KiCad: LLM 連携の成熟度

EasyEDA が大幅にリード:
- MCP ツール数: EasyEDA 72 (267 メソッドブリッジ) vs KiCad 最大 ~20
- 回路図編集: EasyEDA は API ベースで安定. KiCad は公式 Python API なし (S 式直接操作で実験的)
- PCB 配線: EasyEDA はクラウド自動ルーター連携. KiCad は Freerouting 外部ツール経由
- ホットリロード: EasyEDA は WebSocket で即反映. KiCad は再起動必要
- JLCPCB 発注: EasyEDA は一気通貫. KiCad は不可

根本原因: EasyEDA Pro がエクステンション API (WebSocket ブリッジ) を持つのに対し, KiCad は回路図エディタの外部操作 API を提供していない.

### chocofi の KiCad → EasyEDA インポート評価

chocofi のファイル構成:
- `chocofi.sch`: KiCad v4 旧形式 (EESchema Schematic File Version 4)
- `chocofi.kicad_pcb`: KiCad S 式 (version 20211014, KiCad 6 世代)
- `chocofi.kicad_pro`: JSON (KiCad 6+)
- フットプリント: `kbd.pretty/`, `keyswitches.pretty/` (ローカル同梱)

インポート手順:
1. KiCad で `chocofi.kicad_pro` を開く → 回路図が v4→v6 に自動変換される
2. ファイル → プロジェクトをアーカイブで zip 作成
3. EasyEDA Pro で zip をインポート

インポート後に確認が必要な項目:
- 銅箔エリアは再構築される (結果が異なる可能性)
- DRC ルールは保持されない
- LCSC 品番の紐付けは新規に必要

### chocofi の部品構成

| 部品 | フットプリント | 実装方式 |
|---|---|---|
| ダイオード (1N4148) ×42 | `kbd:D3_TH` | スルーホール |
| キースイッチ (Choc) ×36 | `kbd:CherryMX_Choc_1u/1.5u` | スルーホール (手はんだ) |
| Pro Micro ×2 | `kbd:ProMicro_v2_1side` | ピンヘッダ (手はんだ) |
| TRRS ジャック (MJ-4PP-9) ×2 | `kbd:MJ-4PP-9_1side` | スルーホール (手はんだ) |
| OLED ×2 | `kbd:OLED_1side` | ピンヘッダ (手はんだ) |
| リセットスイッチ ×2 | `kbd:ResetSW_1side` | スルーホール (手はんだ) |

JLCPCB PCBA で実装メリットがあるのはダイオード 42 個のみ (SMD SOD-123 に変更すれば LCSC C81598).
基板は Double-sided (1 枚で左右両方に使える設計).

### EasyEDA Pro のセットアップ

- インストール完了: `/opt/apps/easyeda-pro/` に v3.2.149
- symlink: `/usr/local/bin/easyeda-pro`
- インポート済み: `/home/atsushi/Documents/EasyEDA-Pro/projects/choco-stick.eprj2`
- fxtwitter MCP サーバーも導入済み (ツイート調査用)

### 設計上の考慮事項

- TRRS は 4 線 (VCC, GND, データ×2) で使い切り. アナログ信号の左右間転送には別コネクタが必要, または片側 MCU で処理
- 左右対称設計を維持するかどうかが設計判断になる
- ZMK のピンマッピングは `zmk/app/boards/shields/corne/` のデバイスツリーに依存. PCB 変更時は `corne.keymap` の `#ifdef RIGHT_HALF` ブロックも更新が必要 (PLAN.md に既存の方針あり)

### 次のステップ

- [ ] EasyEDA Pro でインポートした chocofi プロジェクトの目視確認 (元 PCB との差分検証)
- [ ] easyeda-copilot エクステンションの導入 (EasyEDA Pro 内)
- [ ] easyeda-copilot MCP サーバーの Claude Code への接続
- [ ] LLM 支援でのピン変更 + アナログジャック追加の試行
