# Bazzite NASの健康監視・状態連携・Recovery実装記録

## 基本情報

- 実施日：2026-08-23
- 分類：Linux / Bazzite / NAS / SMART / systemd / rclone / Recovery
- 状態：実装・異常系検証・正式運用開始済み

## 背景

家庭用NASを常時手動確認するのではなく、通常時は操作不要で、異常が継続している場合だけ気づける構成を作った。監視と判定はAIに依存させず、Linux標準の仕組みと決定論的なスクリプトで完結させ、AIや別端末は生成済みの状態ファイルを読む入口として扱った。

## 対象環境

- OS：Bazzite 44
- ストレージ：USB接続のHDDを使ったSamba共有
- 使用要素：smartmontools / smartctl、shell、systemd、rclone、Samba
- 状態保存：ローカルの軽量ファイルとクラウドストレージ

機器固有のシリアル、ユーザー名、認証情報、実際の保存先識別子は本記録から除外している。

## 実装した構成

状態を次の3種類へ分離した。

- `STATUS`：現在のSMART、容量、Samba、バックアップ、監視状態を集約
- `smart-history.csv`：温度や主要SMART値の軽量な時系列
- `nas-health-warning.md`：異常が継続している間だけ存在し、正常復帰時に削除

SMART取得と状態生成はsystem-level service、クラウド同期とユーザーserviceの状態取得はuser-level serviceへ分離した。監視対象ディスクは再起動で変わり得る`/dev/sdX`ではなく、永続的なby-idパスで指定した。

クラウド同期ではフォルダ全体を同期せず、STATUS、履歴、warningだけを個別に扱った。既存の読み取り専用バックアップ経路を変更せず、状態アップロードは書き込み可能な別経路へ分離した。

## 発生した問題と修正

### rootからuser serviceを直接参照できない

root serviceから`runuser`を経由してuser serviceの状態を取得しようとしたところ、PAM処理とservice hardeningの組み合わせで失敗した。

採用した修正は、user側が小さなstateファイルを定期生成し、root側はそのファイルから許可したキーだけを文字列として読む方式である。root側でuser所有ファイルを`source`せず、権限境界を維持した。

### 古い正常状態が残る

監視処理が停止しても最後の正常値だけが残る問題を避けるため、stateファイルと監視成功時刻に期限を設け、一定時間更新されなければ`STALE`として警告するようにした。

### warning削除で同期処理が失敗する

正常時に存在しないwarningを無条件に削除すると、リモート側の`object not found`でservice全体が失敗した。リモート一覧で存在を確認し、存在する場合だけ対象ファイルを削除する方式へ変更した。

## 異常系の検証

実ディスクを故意に異常化せず、mock SMARTデータを使って次を確認した。

- mock異常が`CRITICAL`と判定される
- warningがローカルとクラウドへ生成される
- mockデータが本番履歴へ混入しない
- 実SMARTへ戻すと正常復帰する
- 正常復帰時にwarningがローカルとクラウドから削除される
- 実履歴には実データだけが追加される

本番経路とテスト経路を分離することで、実ディスクと長期履歴を汚さず異常系を試せた。

## 検証結果

最終検証で、実SMART取得、状態生成、Samba、既存バックアップ、timer永続化、mock異常と正常復帰、クラウド反映、Short self-test開始を確認した。完成時点の総合状態は正常で、監視、履歴、警告、外部参照、復旧資材が一連の運用として接続された。

## Recovery

監視スクリプト、systemd unit、導入・復旧手順、環境確認用の参照情報を小規模なRecovery資材として保存した。復旧処理は現在環境を無条件に上書きせず、内容と対象を確認してから適用する方式とした。

RecoveryにはOAuth token、Client Secret、パスワード、秘密鍵、rclone設定本体等を含めず、認証は復旧先で再設定する。

## 得られた教訓

- 異常判定は決定論的な処理へ任せ、AIは閲覧・説明の入口にする。
- 現在状態、長期履歴、現在の警告を別ファイルにする。
- 正常値だけでなく、更新時刻が古くないかも監視する。
- rootとuserを直接またがず、小さな状態ファイルで責務を分離する。
- クラウドremoteはバックアップ用と状態出力用で権限を分ける。
- 異常系はmockで検証し、本番データ経路を汚さない。
- Recoveryは秘密情報を含めず、認証だけを再設定できる構成にする。

## 関連記録

- `Knowledge/2026-08-23-alienware-alpha-nas-health-monitoring-recovery.md`
- `Public/2026-08-16-bazzite-rclone-google-drive-nas-backup.md`
- `Public/2026-08-23-samba-exfat-vfs-windows-copy-error-public.md`

