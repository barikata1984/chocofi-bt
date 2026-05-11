# CI ビルド調査ログ

GitHub Actions の ZMK ビルドワークフロー関連の記録。append-only。

---

## 2026-05-12: `feat: enable deep sleep` コミットで CI ビルド失敗 → ワークフローピン留めで修正

### 症状

`feat: enable deep sleep for power saving` (`ae997c8`) のプッシュ後、GitHub Actions の `build / Build` ジョブが `Check if building a board without explicit ZMK compat` ステップで失敗。トレースバックは次の通り：

```
Run if ! (grep "CONFIG_ZMK_BOARD_COMPAT=y" "/tmp/.../zephyr/.config" > /dev/null)
Traceback (most recent call last):
  ...
  File "/__w/chocofi-bt/chocofi-bt/zephyr/scripts/west_commands/boards.py", line 87, in do_run
    log.inf(args.format.format(name=board.name, arch=board.arch, ...))
KeyError: 'qualifiers'
Error: Process completed with exit code 1.
```

ファームウェアのビルド (`West Build` ステップ) 自体は成功しており、後続のチェックステップで落ちている。結果として `Rename artifacts` と `Archive` がスキップされ、`.uf2` ファイルが発行されない。

### 原因

ワークフロー `zmkfirmware/zmk/.github/workflows/build-user-config.yml@main` の `Check if building a board without explicit ZMK compat` ステップが、次のコマンドを実行する：

```bash
west boards --board-root ${base}/zmk/app/module \
            --board-root ${base}/zmk/app \
            --board "${original_board}" \
            --format "{qualifiers}"
```

`--format "{qualifiers}"` は新しい Zephyr Hardware Model v2 (hwmv2) で導入された Board オブジェクトの属性に依存する。一方、`config/west.yml` で `zmk@v0.3.0` を固定しているため、その時点の旧 Zephyr が引き込まれ、`Board` オブジェクトに `qualifiers` 属性が存在せず Python の `str.format` が KeyError を投げる。

つまり「ワークフロー (`@main`) が前進した一方で、ユーザー設定 (`west.yml`) は `v0.3.0` に取り残されたバージョン非整合」が根本原因。

### 修正

ZMK 公式ブログ「Pin your ZMK version」(2025-06-20) の推奨通り、**ワークフロー側も同じタグへ揃える**。`.github/workflows/build.yml` を次のように修正：

```yaml
# Before
uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@main

# After
uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3.0
```

`v0.3.0` タグにはワークフローファイルが含まれていることを確認済み (`gh api repos/zmkfirmware/zmk/contents/.github/workflows/build-user-config.yml?ref=v0.3.0`)。

### 注意点

- 過去の成功ビルド `21096070637` (2026-01-17, `zmk version specifed`) は同じ `@main` 参照でも通っていた。これは当時のワークフローが旧 Zephyr API のみを使っていたため。**ワークフローの後方非互換変更が後発的にユーザー設定を壊した** ケース
- 今後 ZMK 本体をバージョンアップする際は、`west.yml` の `revision` と `.github/workflows/build.yml` の `uses:` を **常に同じタグで揃える** こと
