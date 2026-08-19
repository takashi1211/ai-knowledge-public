# BazziteでGoogle DriveをNASへ毎日自動バックアップした記録

> このファイルはブログ記事ではなく、個人情報や秘密情報を除去した第三者向けの作業・検証記録です。

## 基本情報

- 作成日：2026-08-16
- 最終更新日：2026-08-16
- タイトル：BazziteでGoogle DriveをNASへ毎日自動バックアップした記録
- 分類：Linux / Bazzite / NAS / rclone / バックアップ / systemd
- 状態：主要機能は確認済み、一部未検証
- 元になった非公開記録：`2026-08-16-alienware-alpha-google-drive-backup-automation.md`

## 目的

Bazziteを導入した古いPCをNASとして稼働させながら、Google Drive全体を外付けHDDへ一方向コピーし、毎日自動実行する。

安全性を優先して、クラウド側は読み取り専用、バックアップは削除を連鎖させない `copy` 方式とした。

## 背景

BazziteはFedora Atomic系のOSで、一般的なLinux向け手順がそのまま最適とは限らない。また、rcloneをHomebrewで導入している場合、systemdとの組み合わせにも環境固有の癖が出ることがある。

今回の構成では、NAS用外付けHDDがexFATだったため、Google Driveで許容されるファイル名とローカルFS側の命名規則の差も問題になった。

## 使用環境

- 機器：旧世代x86_64 PCを家庭内NASとして再利用
- CPU：第4世代Core i7クラス
- メモリ：16GB
- OS：Bazzite 44
- 使用ソフトウェア：rclone / systemd / Samba
- rclone：1.75.0
- 外付けストレージ：4TB exFAT HDD
- その他の条件：Google Driveは自前OAuthクライアントで接続

## 実施内容

### 1. Google Drive remoteを読み取り専用で作成

```bash
rclone config
```

Google Driveを選び、自前のOAuth Client ID / Client Secretを使用した。

scopeは次を選択。

```text
drive.readonly
```

Google Drive側をrcloneから書き換えられない構成にした。

### 2. Google Cloud側の設定

初回認証ではOAuthアプリのテストユーザー未登録により403になった。

```text
403: access_denied
```

Google Auth Platformで利用アカウントをテストユーザーへ追加した。

さらにGoogle Drive APIが無効だったため、次のエラーも発生した。

```text
SERVICE_DISABLED
Google Drive API ... is disabled
```

Drive APIを有効化した後、次でDrive直下を取得できた。

```bash
rclone lsd gdrive:
```

### 3. 手動コピー

最初はdry-runで対象を確認。

```bash
rclone copy gdrive: <NASのバックアップ先> --dry-run -P
```

問題がないことを確認後、本番実行。

```bash
rclone copy gdrive: <NASのバックアップ先> -P
```

`sync` ではなく `copy` を採用したため、クラウドで削除されたファイルはNASから自動削除されない。

### 4. 実マウント先を確認

当初は `/run/media/...` を想定したが、実際のNAS用HDDはSamba用ディレクトリ配下へマウントされていた。

```bash
lsblk -f
findmnt
```

自動化では「一般的なパス」ではなく実際のマウント先を使う必要がある。

### 5. systemdシステムサービスで失敗

最初はシステムサービスとしてHomebrew版rcloneを起動したが、次のエラーになった。

```text
status=203/EXEC
```

rclone単体は手動で正常だったため、ユーザーsystemdサービスへ切り替えた。

### 6. user systemd service

ユーザーサービスでは、コピー前に外付けHDDが本当にマウントされているか確認するガードを追加した。

```ini
ExecStartPre=/usr/bin/mountpoint -q <NAS HDDのマウント先>
```

rclone本体と設定ファイルはフルパスで指定した。

### 7. exFAT禁止文字を回避

一部ファイルで次のエラーが発生した。

```text
Failed to copy: ... invalid argument
```

原因は、Google Driveでは使える `:` や `?` などの文字がexFATでは使えないことだった。

ローカル側にWindows系ファイルシステム向け文字変換を明示した。

```text
--local-encoding "Slash,LtGt,DoubleQuote,Colon,Question,Asterisk,Pipe,BackSlash,Ctl,RightSpace,RightPeriod,InvalidUtf8,Dot"
```

この変更後、サービスは正常終了した。

### 8. 毎日03:00のtimer

ユーザーtimerを作成。

```ini
[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
Unit=gdrive-backup.service
```

さらに未ログイン状態でもユーザーsystemdを維持できるようlingerを有効化した。

```bash
sudo loginctl enable-linger <user>
```

タイマー一覧で翌日の03:00が次回実行として表示された。

## 結果

- Google Drive→NASの一方向コピー成功。
- Google Drive remoteは読み取り専用。
- `copy` のためクラウド側の削除はNAS側へ連鎖しない。
- user systemd serviceからrcloneを正常実行できた。
- exFAT禁止文字を含むファイル名も文字変換後はコピーできた。
- 毎日03:00のsystemd timerを登録できた。

## 失敗・注意点

### OAuthアプリのテスト状態

設定から約1週間後に次のエラーが発生した。

```text
invalid_grant: maybe token expired?
```

次で再認証して復旧した。

```bash
rclone config reconnect gdrive:
```

OAuthアプリがテスト状態のままでは、長期の無人運用に向かない可能性がある。公開ステータスの確認が必要。

### 同名オブジェクト

Google Drive側に同名オブジェクトがある場合、次の警告が出た。

```text
Duplicate object found in source - ignoring
```

Google Driveと通常のローカルファイルシステムの仕様差による注意点で、クラウドの全オブジェクトを完全に1対1で保存できない場合がある。

## 確認済み

- Bazzite 44でrclone 1.75.0によるGoogle Drive読み取りとコピーが動作。
- user systemd serviceからHomebrew版rcloneを起動できた。
- exFAT禁止文字のエラーは `--local-encoding` で解消。
- service単体は正常終了。
- timerは次回03:00の実行予定を表示。

## 推測

- systemdシステムサービスでの `203/EXEC` は、BazziteのAtomic / SELinux環境とHomebrew prefixの組み合わせが影響した可能性がある。ただし直接原因までは切り分けていない。

## 未検証

- timerが実際に翌03:00に自動実行して成功すること。
- OAuthアプリの長期トークン運用。
- 同名オブジェクトを含むDriveの完全性を担保する別方式。

## 得られた知見

- Bazziteでもrcloneは十分実用できるが、自動化ではHomebrew版の実パスを確認した方がよい。
- 自動化前に外付けHDDの実マウント先を確認することが重要。
- バックアップ先がexFATなら、クラウド側のファイル名がそのまま保存できるとは限らない。
- `drive.readonly` + `rclone copy` は、クラウド原本を守りながらNAS側へ残すバックアップとして安全寄り。
- serviceを先に手動で成功させ、その後timerを追加する方が切り分けしやすい。

## 関連記録

- 元のSession：`2026-08-16-alienware-alpha-google-drive-backup-automation.md`
- 関連Knowledge：`2026-08-16-bazzite-rclone-google-drive-nas-backup.md`
- note記事：note向け完成稿作成済み
- note向け完成稿：`2026-08-16-bazzite-google-drive-nas-auto-backup.md`

## 公開用の処理

- 除去した情報：ユーザー名、メールアドレス、IPアドレス、Google Cloudプロジェクト番号、UUID、個人的なDrive内ファイル名
- 一般化した情報：家庭内の実マウントパス、ユーザー名を含む設定パス
- 公開時の注意点：OAuth Client Secret、rclone.confの中身、アクセストークンは絶対に公開しない
- 外部公開：未実施

## 公開前チェック

- [x] パスワード、APIキー、トークン、秘密鍵を含まない
- [x] 個人名、ユーザー名、メールアドレスを含まない
- [x] IPアドレス、UUID、家庭内ネットワーク情報を含まない
- [x] 公開不要な個人的事情を含まない
- [x] 第三者の個人情報を含まない
- [x] 推測を確認済みの事実として書いていない
- [x] 未検証を明記した
- [x] 元記録との関係を記載した
- [x] 読み物へ過度に再構成せず、作業記録の役割を保った

## 変更履歴

- 2026-08-16：初版作成
