# main 差分最小化 + local build 成功の実行計画（更新版）

## 現時点の確定事項
- `build.yaml` 準拠条件でローカルビルド成功を再確認済み。
  - left: `SHIELD=sweep_left` + `SNIPPET=studio-rpc-usb-uart` + `CONFIG_ZMK_STUDIO=y`
  - right: `SHIELD=sweep_right`
- `zmk@main` + `board: nice_nano` で左右とも成功。
- `config/west.yml` は `origin/main` 相当（`halfdane` / `cirque-input-module` 含む）でも成功。
- `board: nice_nano_v2` は `zmk@main` でも失敗（`Invalid BOARD`）。

## main 差分最小の方針
1. `origin/main` と同一にできる差分は戻す。
2. ビルド成立に必須な差分だけ残す。
3. 判定は `build.yaml` 準拠コマンドのみで行う（非準拠コマンドは補助情報扱い）。

## 残すべき差分（現時点の候補）
- `build.yaml`
  - `board: nice_nano_v2` -> `board: nice_nano` は必須。
- `boards/shields/sweep/sweep_left.overlay`
  - `mipi_dbi` 形式への移行は必須。
- `boards/shields/sweep/sweep_right.overlay`
  - `dr-gpios` -> `data-ready-gpios` は必須。
- `config/sweep_left.conf`
  - `LV_USE_IMG/LV_USE_PNG` 削除と `CONFIG_ZMK_WIDGET_PERIPHERAL_STATUS=n` は必須。

## main に戻せる差分（現時点の候補）
- `config/west.yml`
  - `origin/main` 相当に戻してもビルド成功。
- `config/sweep.conf`
  - `CONFIG_WARN_DEPRECATED=n` は不要（削除可）。

## 実行ステップ（完了）
1. `config/west.yml` と `config/sweep.conf` を `origin/main` 相当に固定。
2. その状態で `build.yaml` 準拠の左右ビルドを再実行し成功。
3. `git diff origin/main` で必須差分のみが残る状態を確認。
4. `report.md` へ検証結果を追記。
