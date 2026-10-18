### Dropbox子プロセスの親終了とsystemdの追跡方式を調整した復旧記録

Arch Linux上で発生したDropboxの起動・終了反復を調査し、systemdのサービス追跡方式を修正した実測記録です。

**環境と症状**

systemd 262、Dropboxのversioned binaryは274.4.4905。user serviceは`Type=simple`で起動していました。約6秒でMainPIDがSIGKILL終了し、vendor設定の`Restart=on-failure`によって起動を繰り返していました。`KillMode=process`のもとで残留プロセス警告も記録され、高いCPU負荷が続いていました。

調査中は`Restart=no`と`KillMode=control-group`を設定し、自動起動の重複を抑止しました。単一起動でも約6秒のSIGKILLは再現しましたが、自動再起動は発生しませんでした。

**送信元の特定**

bpftraceの`signal_generate`観測で、Dropbox起動系列の子プロセスが親launcherへSIGKILLを送信していたことを確認しました。親終了後もversioned childは生存し、Dropboxのpid fileもchildを指していました。

この挙動はlauncherからchildへの引き継ぎと整合します。ただし、vendorバイナリ内部の設計意図を直接確認したものではありません。

**修正**

Dropbox専用user serviceのdrop-inに以下を追加しました。

```ini
[Service]
ExitType=cgroup
ExecReload=
```

`ExitType=cgroup`により、親が終了してもcgroup内のchildが生存する間はサービスをactiveとして維持できます。

vendorのreload処理は`kill -HUP $MAINPID`でした。引き継ぎ後にMainPIDが0になるため、`ExecReload=`で無効化しました。既存の`Restart=no`と`KillMode=control-group`は維持しています。

**検証結果**

- 16分以上の継続稼働、`NRestarts=0`
- OOMカウンター0、定常CPU約0.3〜0.8%
- Dropboxがidle／「最新の状態」を表示
- 無害な一時markerの作成・削除後も同期完了表示を確認

別端末やDropbox Webによる独立した同期read-backは未実施です。MainPIDが0、または過去の`Result=signal`が残っていても、それだけで停止と判断せず、unit状態・child・cgroupを合わせて確認する必要があります。

**自動起動と限界**

Lingerが有効な環境でGUI開始前の起動を避けるため、自動起動を`default.target`から`graphical-session.target`へ変更し、XDG autostartとの重複を抑止しました。

設定時には既存セッションとDropboxが稼働していたため、新規セッションによる自動起動は未実測です。起動後のXDG entry再生成も未確認です。復旧の技術判定はPASS、自動起動実測の判定はINCOMPLETEとして区別しています。

この設定は当該環境の観測に基づくもので、Dropbox全環境への一律適用を検証したものではありません。公開用にhostname、ユーザー名、PID、内部記録リンク等を除去・一般化しました。

参照：[systemd v262 service設定](https://github.com/systemd/systemd/blob/v262/man/systemd.service.xml)、[Linux signal tracepoint](https://github.com/torvalds/linux/blob/master/include/trace/events/signal.h)、[Dropbox同期状態の説明](https://help.dropbox.com/sync/check-sync-status)

