### 以下後ほど docs/ へ移動予定

# 設計書：git_rename.sh

## 概要：
git_rename.sh は、指定した文字列を Git リモート URL から一括で置換し、
複数プロジェクトのリモート設定を同時に更新する自動化スクリプトである。
これにより手作業の煩雑さを排し、高速かつ確実にリモート URL のリネームを実現する。

## 目的：
- ローカルにクローンされた複数の Git リポジトリに対し、旧リポジトリ名を新リポジトリ名へ一括置換

- 作業ミスを排し、統一的なログ出力で変更状況を可視化

- 短時間での大量リポジトリ更新を可能とする

## 機能要件:
| ID  | 要件                                        |
| --- | ----------------------------------------- |
| FR1 | スクリプト引数として `<old>` `<new>` を受け取ること        |
| FR2 | オプションでベースディレクトリを指定可能であること                 |
| FR3 | 指定ディレクトリ配下の各リポジトリを走査し、`.git` 有無を判定すること    |
| FR4 | 現行の `origin` URL を取得し、文字列置換後の URL へ更新すること |
| FR5 | 変更前・変更後の URL をログに出力すること                   |
| FR6 | 変更不要時には「変更不要」としてスキップを表示すること               |

## 非機能要件:
- 可読性：シンプルな構造、適切なコメント記述

- 拡張性：置換ルールの汎用性（部分一致での置換）

- 安全性：set -euo pipefail を導入し、未定義変数／パイプ失敗で即時停止

- 移植性：Linux, macOS 両対応

## 操作フロー:
### 1. **引数チェック**
- 引数が 2〜3 個でない場合は Usage を出力して終了

### 2. **変数設定**
- OLD, NEW, BASE_DIR を設定

### 3. **ベースディレクトリ配下を走査**
- find $BASE_DIR -maxdepth 2 -type d でサブディレクトリを取得

### 4. **各ディレクトリ内で .git 存在確認**
- 存在すれば git remote get-url origin で現行 URL を取得

### 5. **文字列置換**
- Bash 変数展開により CURRENT_URL の $OLD → $NEW

### 6. **更新判定**
- CURRENT_URL != NEW_URL なら git remote set-url origin NEW_URL 実行

### 7. **ログに「Before / After」を出力**
- 等しければ「変更不要」を出力

### 8. **完了メッセージ出力**



## 入出力仕様：
#### **入力**
- コマンドライン引数：
  - 置換前文字列（例：Hideyuki-T）
  - 置換後文字列（例：Hideyuki-Tamai）
  - （省略可）ベースディレクトリ（デフォルト：カレントディレクトリ）

#### **出力**
- 標準出力に進捗ログを出力：
  - 更新対象リポジトリ名
  - Before: <old_url>
  - After: <new_url>
  - 変更不要リポジトリ名

##  エラーハンドリング：
- 未定義引数：Usage を表示して exit 1
- git remote get-url や set-url が失敗した場合、set -e により即時中断
- 権限不足やネットワーク障害時は Bash のエラーメッセージを参照

## セキュリティ・運用考慮：
- リモート URL を置換するため、プライベートトークンや機密情報は扱わない
- 万が一のため、事前に git remote -v で現状をバックアップ的に記録推奨
- 実行前に --dry-run モードを追加実装し、変更内容のみを表示する拡張余地あり

## テストケース：
| No. | シナリオ                | 期待結果              |
| --- | ------------------- | ----------------- |
| TC1 | 引数不足                | Usage 表示・異常終了     |
| TC2 | 非 Git リポジトリディレクトリ混在 | 変更対象外として無視        |
| TC3 | 旧文字列含むリモート URL が存在  | URL 置換後に正常更新・ログ出力 |
| TC4 | 旧文字列含まないリモート URL のみ | 変更不要としてログ出力       |
| TC5 | ベースディレクトリ省略         | カレントディレクトリ走査      |

