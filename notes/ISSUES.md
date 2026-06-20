# 未解決の技術課題

## バージョン依存リスク

### ZMK v0.3.0 ピン留めの陳腐化

`config/west.yml` で `zmk@v0.3.0` を固定しているため, 本家側で進む API・Kconfig 改名(例: `CONFIG_ZMK_POINTING` の整備状況, `require-prior-idle-ms` の挙動更新)の恩恵を受けられない. アップデート時には `corne.keymap` の `&mt` / `&mmv` 周りと, `CONFIG_ZMK_DISPLAY_BLANK_ON_IDLE` 等のディスプレイ系 Kconfig の差分確認が必要.

なお, `west.yml` で ZMK 本体をピン留めする際は GitHub Actions ワークフロー (`.github/workflows/build.yml`) の `uses:` も同じタグへ揃える必要がある. `@main` のままにすると, 後方非互換変更(例: `west boards --format "{qualifiers}"` の追加)で `Check if building a board without explicit ZMK compat` ステップが KeyError で落ちる.
