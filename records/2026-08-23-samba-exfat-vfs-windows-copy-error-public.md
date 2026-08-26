# exFATのSamba共有でWindowsのファイルコピーがERROR 50になる問題を解決した記録

> このファイルはブログ記事ではなく、個人情報や秘密情報を除去した第三者向けの作業・検証記録です。

## 基本情報

- 作成日：2026-08-23
- 最終更新日：2026-08-23
- 分類：Linux / Samba / NAS / exFAT / Windows / VFS
- 状態：回避策は実機確認済み、原因モジュールの個別比較は未検証
- 元記録：`Sessions/2026-08-23-alienware-alpha-samba-exfat-windows-copy-fix.md`

## 概要

Bazzite上でexFAT外付けHDDをSamba共有し、Windowsから利用していた。共有の一覧表示、フォルダ作成、小さいテキストファイルの読み書きはできる一方、約698MBの動画ファイルをコピーすると失敗した。

`robocopy` では次のエラーになった。

```text
エラー 50 (0x00000032)
この要求はサポートされていません。
```

Sambaのglobal設定にあった `vfs objects = catia fruit streams_xattr` がexFAT上の共有にも継承されていた。対象共有だけVFS継承を無効化し、Samba再起動とWindows側再接続を行ったところ、同じファイルをエクスプローラーから正常にコピーできた。

## 環境

- OS：Bazzite
- 共有：Samba
- 保存先：4TB exFAT外付けHDD
- クライアント：Windows 10
- 接続：SMB共有をネットワークドライブへ割り当て

## 症状

- SMB共有へ接続できる
- 既存ファイルを一覧表示できる
- フォルダを作成できる
- 小さいテキストファイルを作成・読み戻しできる
- 約698MBの通常ファイルはExplorer、`copy`、PowerShell、`robocopy` でコピーできない
- Explorerではコピー元が見つからない趣旨の表示
- `robocopy` ではエラー50

コピー元ファイルは実在し、再解析ポイントでもなかった。

## Samba設定で確認した点

`testparm -s` で、globalに次の設定があることを確認した。

```ini
[global]
    fruit:model = MacSamba
    fruit:aapl = yes
    vfs objects = catia fruit streams_xattr
```

対象共有には個別の `vfs objects` がなく、global設定を継承していた。

保存先はexFATであるため、xattrを利用するVFSモジュールとの機能差が疑われた。

## 実施した修正

対象共有に空の `vfs objects` を明示した。

```ini
[nas]
    path = <exFAT HDDの共有パス>
    read only = no
    valid users = <user>
    force user = <user>
    vfs objects =
```

これにより、対象共有だけglobalの `catia fruit streams_xattr` を継承しないようにした。

設定変更後は次を実施した。

1. `testparm` で設定ファイルの正常読込みを確認
2. Sambaサービスを再起動
3. Windows側の古いネットワークドライブ接続を削除
4. UNCパスをネットワークドライブとして再割り当て
5. 修正前に失敗した同じファイルを再コピー

## 結果

約698MBの動画ファイルをWindowsエクスプローラーから通常のコピー＆ペーストで保存できた。

問題解決のためにexFAT HDDを再フォーマットする必要はなかった。

## 確認済み

- 認証や共有の読み書き権限は修正前から成立していた。
- `robocopy` のエラー50はVFS継承を外す前に再現した。
- 対象共有でVFS継承を外した後、同じファイルがコピー成功した。
- Windowsではネットワークドライブ経由の通常操作が可能になった。

## 原因についての判断

exFAT上の共有へglobalの `catia fruit streams_xattr` が適用されていたことが原因である可能性が高い。特に `streams_xattr` とexFATの機能差が主因だった可能性がある。

ただし、3つのVFSモジュールを一つずつ比較していないため、`streams_xattr` 単独を原因とは断定しない。

## 注意点

fruit有効共有と無効共有が同じSambaサーバー内に混在すると、Macクライアントでの併用に関する警告が出る場合がある。

```text
WARNING: some services use vfs_fruit, others don't. Mounting them in conjunction on OS X clients results in undefined behaviour.
```

今回確認したのはWindowsから対象共有を利用する構成である。Macからfruit有効・無効の共有を同時利用する場合は別途検証が必要。

また、Windowsの「ネットワーク」一覧に共有が見えない問題は、IPアドレスを指定したUNCパスで到達できるなら、今回のVFS問題とは分けて考えられる。固定のネットワークドライブへ割り当てれば、探索表示に依存せず利用できる。

## 得られた知見

- 小さいファイルを書けることだけでは、SMB経由の通常ファイルコピー全体が正常とは判断できない。
- `ERROR 50 / この要求はサポートされていません` では、権限だけでなくSamba VFSと保存先ファイルシステムの互換性を確認する。
- `testparm -s` では共有個別設定だけでなく、globalから継承される `vfs objects` に注意する。
- exFATを維持したまま、対象共有だけ不要なVFS継承を外すことで解決できる場合がある。

## 関連記録

- 元Session：`Sessions/2026-08-23-alienware-alpha-samba-exfat-windows-copy-fix.md`
- 関連Knowledge：`Knowledge/2026-08-23-samba-exfat-vfs-windows-copy-error.md`
- 関連構成：Bazzite上のrcloneによるGoogle Drive→exFAT NASバックアップ

## 公開用の処理

- 除去・一般化：ユーザー名、IPアドレス、Windowsアカウント名、家庭内の固有パス
- 秘密情報：パスワード、認証情報、トークンは含めていない
- 外部公開：未実施

## 変更履歴

- 2026-08-23：初版作成
