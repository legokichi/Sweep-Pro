# fix/zephyr41-local-build-no-keymap: 変更サマリ

## 目的
- Zephyr 4.1 環境で `Sweep-Pro` をローカルビルド可能にする。
- キーマップ変更を主目的にせず、ビルド互換性の確保を優先する。
- `PR #1` の内容（ZMK revision 固定と board 名調整）を現行ブランチへ統合する。

## 変更ファイルと意図
| ファイル | 変更内容 | 意図 |
|---|---|---|
| `config/west.yml` | `zmk` の revision を `c06fa48...` に固定、`halfdane` 関連モジュール参照を削除 | Zephyr 4.1 での不整合回避、追加モジュール由来の競合回避、再現性向上 |
| `build.yaml` | `board: nice_nano_v2` → `board: nice_nano` | 固定 revision で有効な board 名へ合わせる |
| `boards/shields/sweep/sweep_left.overlay` | e-paper 定義を `&spi0` 直下から `mipi_dbi` ノード構成へ移行、`spi-max-frequency` → `mipi-max-frequency`、`dc-gpios` は `GPIO_ACTIVE_LOW` | Zephyr 4.1 の display/mipi_dbi バインディングへ追従し、電子ペーパー初期化失敗を回避 |
| `boards/shields/sweep/sweep_right.overlay` | `dr-gpios` → `data-ready-gpios`（`GPIO_ACTIVE_LOW`） | Pinnacle ドライバのプロパティ名変更に追従 |
| `config/sweep.conf` | `CONFIG_WARN_DEPRECATED=n` 追加 | 互換移行中の非致命な deprecate 警告を抑制 |
| `config/sweep_left.conf` | `LV_USE_IMG`, `LV_USE_PNG` 削除、`CONFIG_ZMK_WIDGET_PERIPHERAL_STATUS=n` | Zephyr 4.1 + 現設定での不要/不整合オプション整理 |

## `PR #1` との統合内容
- 取り込み済みの意図:
  - ZMK revision 固定
  - `nice_nano` board への切替
  - 追加モジュール由来の衝突回避
- 本ブランチでは `PR #1` より強く、`halfdane` 系モジュール参照自体を `west.yml` から外している。

## ローカル検証結果
- 成功したビルド例:
  - `west build -p -d build/left_boardroot -s zmk/app -b nice_nano -- -DSHIELD=sweep_left -DBOARD_ROOT=$PWD -DZMK_CONFIG=$PWD/config -DKEYMAP_FILE=$PWD/boards/shields/sweep/sweep.keymap`
  - `west build -p -d build/right_boardroot -s zmk/app -b nice_nano -- -DSHIELD=sweep_right -DBOARD_ROOT=$PWD -DZMK_CONFIG=$PWD/config -DKEYMAP_FILE=$PWD/boards/shields/sweep/sweep.keymap`
- 生成物:
  - `build/left_boardroot/zephyr/zmk.uf2`
  - `build/right_boardroot/zephyr/zmk.uf2`

## 補足
- ローカルでは `BOARD_ROOT` と `KEYMAP_FILE` を明示すると安定して再現しやすい。
- 右手ビルド時の一部 Kconfig warning（split 側で無効化される設定）は観測されるが、ビルド完了は確認済み。
