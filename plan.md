# main 差分最小化 + local build 成功の実行計画（更新版）

## 現時点の確定事項
- `build.yaml` 準拠条件でローカルビルド成功を再確認済み。
  - left: `BOARD=nice_nano//zmk` + `SHIELD=sweep_left` + `SNIPPET=studio-rpc-usb-uart` + `CONFIG_ZMK_STUDIO=y`
  - right: `BOARD=nice_nano//zmk` + `SHIELD=sweep_right`
- `zmk@main` + `board: nice_nano` で左右とも成功。
- 現在の CI 失敗は `cirque-input-module` と Zephyr 側 Pinnacle 実装の重複が原因。
  - devicetree binding 重複（`cirque,pinnacle`）
  - Kconfig 重複（`INPUT_PINNACLE`）
- `board: nice_nano_v2` は `zmk@main` でも失敗（`Invalid BOARD`）。

## main 差分最小の方針
1. `origin/main` と同一にできる差分は戻す。
2. ビルド成立に必須な差分だけ残す。
3. 判定は `build.yaml` 準拠コマンドのみで行う（非準拠コマンドは補助情報扱い）。

## 残すべき差分（現時点の候補）
- `config/west.yml`
  - `cirque-input-module` の削除は CI 通過のため必須。
- `build.yaml`
  - `board: nice_nano_v2` -> `board: nice_nano//zmk` は必須。
- `boards/shields/sweep/sweep_left.overlay`
  - `mipi_dbi` 形式への移行は必須。
- `boards/shields/sweep/sweep_right.overlay`
  - `dr-gpios` -> `data-ready-gpios` は必須。
- `config/sweep_left.conf`
  - `LV_USE_IMG/LV_USE_PNG` 削除と `CONFIG_ZMK_WIDGET_PERIPHERAL_STATUS=n` は必須。

## main に戻せる差分（現時点の候補）
- `config/sweep.conf`
  - `CONFIG_WARN_DEPRECATED=n` は不要（削除可）。

## 実行ステップ（更新）
1. `config/west.yml` から `cirque-input-module` のみ削除し、他は維持。
2. `build.yaml` 準拠の左右ビルドを再実行して成功を確認。
3. 変更を commit/push して PR CI の結果を確認。
