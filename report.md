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

## `PR #1` 作業記録（report2 統合）
### 作業目的
- `AGENTS.md`（Contributor Guide）を追加すること。
- ローカルで ZMK ファームウェアをビルド可能にすること。
- Zephyr/ZMK `main` 系の仕様変更に追従し、ビルドエラーを解消すること。
- keymap 変更を含めない修正ブランチを作成して push すること。

### 実施内容
1. `AGENTS.md` を作成（`Repository Guidelines`、構成・コマンド・テスト・PR方針を記載）。
2. ローカルビルド環境を整備。
   - `west` / `ninja` を導入
   - Zephyr Python 依存を導入
   - Zephyr SDK を導入
   - `protoc` 不足を `grpcio-tools` で補完
3. ZMK ワークスペースを別ディレクトリで作成し、`west update` を実行。
4. ビルド失敗要因を調査し、以下の互換修正を適用。
   - `boards/shields/sweep/sweep_left.overlay`
     - SSD16xx 定義を Zephyr 4.1 の `mipi_dbi` ラッパー形式へ移行
     - `dc-gpios` 極性を現行仕様に合わせて調整
   - `boards/shields/sweep/sweep_right.overlay`
     - `dr-gpios` を `data-ready-gpios` に変更（現行 binding 対応）
   - `config/sweep_left.conf`
     - 現行で未定義となる LVGL 設定を整理
     - `CONFIG_ZMK_WIDGET_PERIPHERAL_STATUS=n` を設定
   - `config/sweep.conf`
     - `CONFIG_WARN_DEPRECATED=n` を追加
5. ビルド検証。
   - 左右とも `zmk.uf2` 生成まで成功。
6. `keymap` 変更を除外した修正ブランチを作成して push。
   - branch: `fix/zephyr41-local-build-no-keymap`
   - keymap を含まない変更のみで commit
   - `origin/fix/zephyr41-local-build-no-keymap` へ push

### 備考
- 当時の PR 作成 URL:
  - `https://github.com/legokichi/Sweep-Pro/pull/new/fix/zephyr41-local-build-no-keymap`
- 本 `report.md` は、その後の統合作業（`PR #1` 取り込み・追加修正）を反映した最新版。

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
