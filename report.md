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

## 逐次作業ログ（main 差分最小化）

### 2026-03-05 07:50:29 UTC
- 目的を「`origin/main` 差分の最小化 + ローカル左右ビルド成功」に再設定。
- 基準状態の再検証を実施（`nice_nano` 固定、現行 `west.yml`/overlay/conf のまま）。
  - `baseline_right`: 成功（`build-baseline-right/zephyr/zmk.uf2` 生成）
  - `baseline_left(studio)`: 成功（`build-baseline-left/zephyr/zmk.uf2` 生成）
- 次ステップ:
  1. `config/sweep.conf` の `CONFIG_WARN_DEPRECATED=n` を戻して再ビルド
  2. `config/sweep_left.conf` の差分を戻して再ビルド
  3. `sweep_right.overlay` / `sweep_left.overlay` を順に戻して必要性を判定
  4. `build.yaml`/`west.yml` の最小必要差分を再確認

### 2026-03-05 07:52:32 UTC
- `test1`: `config/sweep.conf` の `CONFIG_WARN_DEPRECATED=n` を削除して再検証。
- 手順補足:
  - 初回は `west zephyr-export` 未実行で失敗（手順ミス）。
  - `zephyr-export` 後に再実行し、評価を確定。
- 結果:
  - `right`: 成功
  - `left(studio)`: 成功
- 判定:
  - `CONFIG_WARN_DEPRECATED=n` は **ビルド成立に必須ではない**（差分削減可能）。

### 2026-03-05 07:54:17 UTC
- `test2`: `config/sweep_left.conf` を `origin/main` 相当に戻して再検証。
- 結果:
  - `right`: 成功
  - `left(studio)`: 失敗
- 主要エラー:
  - `LV_USE_IMG` / `LV_USE_PNG` が未定義シンボルとして警告
  - `error: Aborting due to Kconfig warnings`
- 判定:
  - `config/sweep_left.conf` の差分（`LV_USE_IMG/LV_USE_PNG` 削除、`CONFIG_ZMK_WIDGET_PERIPHERAL_STATUS=n`）は **必須**。

### 2026-03-05 07:55:39 UTC
- `test3`: `sweep_right.overlay` の `data-ready-gpios` を `origin/main` の `dr-gpios` に戻して再検証。
- 結果:
  - `right`: 失敗
  - `left(studio)`: 成功
- 主要エラー:
  - `dr-gpios ... is not declared ... cirque,pinnacle-spi.yaml`
- 判定:
  - `boards/shields/sweep/sweep_right.overlay` の `data-ready-gpios` 変更は **必須**。

### 2026-03-05 07:57:15 UTC
- `test4`: `sweep_left.overlay` を `origin/main` 相当に戻して再検証。
- 結果:
  - `right`: 成功
  - `left(studio)`: 失敗
- 主要エラー:
  - `ssd16xx.c` コンパイル時に `mipi_max_frequency` 関連で失敗
  - 旧 overlay 形式（`&spi0` 直下の `display@0` + `spi-max-frequency`）が Zephyr 4.1 構成と不整合
- 判定:
  - `boards/shields/sweep/sweep_left.overlay` の `mipi_dbi` 形式への変更は **必須**。

### 2026-03-05 07:58:01 UTC
- `test5`: board 指定を `nice_nano_v2` として左右を再検証（`zmk` は `c06fa48...` pin のまま）。
- 結果:
  - `right`: 失敗
  - `left(studio)`: 失敗
- 主要エラー:
  - `Invalid BOARD`
  - `No board named 'nice_nano_v2' found.`
- 判定:
  - 現行 pin 構成では `build.yaml` の board は `nice_nano` が **必須**（`nice_nano_v2` は不可）。

### 2026-03-05 07:59:40 UTC
- `test6`: `config/west.yml` の `zmk` revision を `main` に戻し、board は `nice_nano` のまま再検証。
- 結果:
  - `right`: 成功
  - `left(studio)`: 成功
- 判定:
  - 現時点では `zmk` pin（`c06fa48...`）は **必須ではない**。
  - `config/west.yml` は `main` 追従へ寄せて差分削減可能。

### 2026-03-05 08:01:10 UTC
- `test7`: `zmk` を `main` のまま board を `nice_nano_v2` へ変更して再検証。
- 結果:
  - `right`: 失敗
  - `left(studio)`: 失敗
- 主要エラー:
  - `Invalid BOARD`
  - `No board named 'nice_nano_v2' found.`
- 判定:
  - `zmk main` でも `nice_nano_v2` はこの構成で利用不可。`build.yaml` は `nice_nano` 維持が必要。

### 2026-03-05 08:03:20 UTC
- `test8`: `config/west.yml` を `origin/main` 相当（`halfdane` + `cirque-input-module` 含む）にして再検証。
- 実行時注意:
  - `-DSHIELD=\"sweep_left nice_view_adapter nice_view\"` で試すと `undefined node label 'nice_view_spi'` で失敗。
  - これは `build.yaml` 非準拠の手動条件。
- 判定:
  - この失敗は CI 条件の代替にならないため、以後は `build.yaml` 準拠コマンドで評価する。

### 2026-03-05 08:05:45 UTC
- `test9`: `cirque-input-module` を一度除去し、`build.yaml` 準拠コマンドで再評価。
  - 左手: `-DSHIELD=sweep_left -DSNIPPET=studio-rpc-usb-uart -DCONFIG_ZMK_STUDIO=y`
  - 右手: `-DSHIELD=sweep_right`
- 結果:
  - `left (CI相当)`: 成功
  - `right`: 成功
- 判定:
  - CI 準拠ではビルド成功。

### 2026-03-05 08:07:33 UTC
- `test10`: `cirque-input-module` を再追加し、`config/west.yml` を再び `origin/main` 相当へ戻して `build.yaml` 準拠で最終確認。
- 結果:
  - `left (CI相当)`: 成功（`build-ci-test10-left/zephyr/zmk.uf2`）
  - `right`: 成功（`build-ci-test10-right/zephyr/zmk.uf2`）
- 判定:
  - `config/west.yml` の `halfdane`/`cirque-input-module` 削除は **必須ではない**。
  - `west.yml` は `origin/main` と同等に維持可能。

### 2026-03-05 08:08:56 UTC
- `plan.md` を更新し、`build.yaml` 準拠で判定する方針を明文化。
- `origin/main` との差分確認:
  - 依然として必須候補は `build.yaml`, `sweep_left/right.overlay`, `config/sweep_left.conf`。
  - `config/west.yml` と `config/sweep.conf` は `origin/main` 相当に戻せることを確認済み。

### 2026-03-05 08:29:00 UTC
- `config/west.yml` と `config/sweep.conf` を `origin/main` 相当に戻した状態で最終再検証。
- 実施:
  - 右手: `build-final-right` (`-DSHIELD=sweep_right`)
  - 左手(CI相当): `build-final-left` (`-DSHIELD=sweep_left -DSNIPPET=studio-rpc-usb-uart -DCONFIG_ZMK_STUDIO=y`)
- 結果:
  - `right`: 成功（`build-final-right/zephyr/zmk.uf2`）
  - `left (CI相当)`: 成功（`build-final-left/zephyr/zmk.uf2`）
- 判定:
  - `config/west.yml` / `config/sweep.conf` のブランチ独自変更は不要。
  - `main` 差分最小化の観点で、必須候補は `build.yaml`, `sweep_left.overlay`, `sweep_right.overlay`, `config/sweep_left.conf` に収束。

### 2026-03-05 08:29:59 UTC
- 差分最小化のため、ビルド非本質な `orig.l/*` と `orig.r/*`（UF2/INDEX/INFO）を Git 追跡から削除。
- 目的:
  - `origin/main` との差分からローカル検証用生成物を除外し、PR差分を機能変更に限定する。

### 2026-03-05 08:30:55 UTC
- `plan.md` を完了状態へ更新。
- 現在の到達点:
  - `main` 差分（機能変更）は `build.yaml`, `sweep_left/right.overlay`, `config/sweep_left.conf` に集約。
  - `config/west.yml` と `config/sweep.conf` は `main` 同等化済み。

### 2026-03-05 17:15:49 UTC
- PR CI 失敗の再調査（GitHub Actions run: `22709701323`, `22709699813`）。
- 失敗要因:
  - `right` ジョブ: `cirque,pinnacle-i2c.yaml` の `compatible: cirque,pinnacle` 重複定義。
  - `left` ジョブ: `INPUT_PINNACLE` の Kconfig 重複定義。
- 判定:
  - `config/west.yml` の `cirque-input-module` が、現在の Zephyr 側実装と衝突している。
- 対応:
  - `config/west.yml` から `cirque-input-module` のみ削除（`zmk-input-processors` は維持）。
- ローカル再検証（Docker + CI 相当引数）:
  - `right`: 成功（`.local-zmk/build-cifix-right/zephyr/zmk.uf2`）
    - `-DSHIELD=sweep_right`
  - `left`: 成功（`.local-zmk/build-cifix-left/zephyr/zmk.uf2`）
    - `-DSHIELD=sweep_left -DSNIPPET=studio-rpc-usb-uart -DCONFIG_ZMK_STUDIO=y`
- 補足:
  - 左手は `studio-rpc-usb-uart` を `SHIELD` に指定すると失敗するため、`build.yaml` 通り `SNIPPET` 指定で評価。

### 2026-03-05 17:25:33 UTC
- 変更を commit/push:
  - commit: `4f09ae8` (`fix(ci): drop cirque-input-module to resolve pinnacle conflicts`)
  - branch: `fix/zephyr41-local-build-no-keymap`
- CI 監視結果:
  - GitHub 側で push event は受理（`2026-03-05T17:16:46Z`）。
  - しかし `head_sha=4f09ae8...` の check suite / workflow run が生成されず、`gh run list` でも新規 run が出現しない状態。
- 手動起動試行:
  - `gh workflow run build.yml --ref fix/zephyr41-local-build-no-keymap` を2回実行。
  - いずれも `HTTP 500: Failed to run workflow dispatch` で失敗。
- 現時点の結論:
  - ローカル（CI相当）では左右とも成功。
  - GitHub Actions はリポジトリ側で run 生成に失敗しており、CI pass/fail 判定は未取得。
