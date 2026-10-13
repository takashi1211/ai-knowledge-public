# Codex Remote ControlのGUI依存をsystemd user serviceで解消した記録

2026年10月13日、Arch Linux環境で、GUI終了後にCodex Remote Controlへ接続できなくなる問題を調査・修正した。GUI中、ログアウト後TTY、GUI未起動の再起動後TTYで、iPhoneからの実接続を確認した記録である。

## 環境と症状

対象はDell OptiPlex 3050 Micro、Arch Linux、Hyprland + Caelestia。通常は`multi-user.target`で起動し、GUIは必要なときだけ起動する構成だった。

Codex CLIは0.162.1、managed daemonは0.161.0。Remote Controlはsystemd user serviceとtimerを使用し、lingerも有効だった。

GUIからログアウトしてTTYへ戻ると、iPhoneから接続できなくなった。timerはactive/waiting、failed user unitは0件で、journalにも起動成功が記録されていた。

serviceは`Type=oneshot`で、成功終了後に`inactive/dead`となっていた。この表示自体は異常ではなく、launcherの成功と遠隔接続の成功は別に確認する必要があった。

## 原因を絞り込んだEvidence

managed daemonは、Hyprlandから起動したterminal内のCodex CLIを親に持ち、Hyprland unitのcgroupに所属していた。同unitは`KillMode=control-group`だった。

GUI終了後の実接続失敗、旧daemon PIDの消失、daemonの所属cgroupが整合し、GUI unitの停止にdaemonの寿命が結び付いている構造が主因として強く裏付けられた。ただし、旧daemonの正確な終了signalやexit statusは直接取得していない。

Remote Control service側に追加した`KillMode=process`だけでは、既にGUI cgroup内で動くdaemonを別のcgroupへ移せなかった。

## bootstrap単独では解決しなかった

CLI経由の操作はmanaged package更新を誘発し得ることが隔離検証で分かったため、既存0.161.0 binaryを使用した。

```bash
~/.codex/packages/app-server-daemon/current/bin/codex app-server daemon bootstrap --remote-control
```

GUI内から実行するとdaemonは再起動したが、新processもHyprland cgroup内に残った。

この環境では、detached processとして起動することと、systemd cgroupをGUIから独立させることは別だった。

実行中に作業接続が切れ、完了状態が一時不明になったため、重複実行せず、再接続後にPID・cgroup・生成物を読み取り確認した。

## 有効だった修正

既存のsystemd user serviceにdrop-inを追加し、managed binaryをuser manager経由で起動する方法へ変更した。

最初の再配置には`app-server daemon restart`を使用した。旧daemonは60秒のgrace後に強制停止され、一時的に作業接続が切れた。その後、新daemonはRemote Control serviceの独立したcgroupで起動した。

恒常的な起動コマンドは、同じbinaryの次のコマンドへ変更した。

```bash
~/.codex/packages/app-server-daemon/current/bin/codex app-server daemon start
```

timerを再開すると、複数周期で`alreadyRunning`とexit 0が返り、daemon PIDは維持された。

systemdにはleft-over process warningが残ったが、launcher成功、daemon継続、failed unit 0件と両立していた。warning単独で障害とは判定しなかった。

## 実機検証と結果

| 条件 | iPhoneからの接続 |
| --- | --- |
| 修正後のGUI起動中 | 成功 |
| GUIログアウト後のTTY | 成功 |
| 再起動後、GUIを一度も起動しないTTY | 成功 |

daemonがGUI cgroup外で稼働していること、timer正常、linger有効、failed user unit 0件も確認した。SSH、Samba、Jellyfin、Cockpit、NetworkManagerはactiveだった。

人間による実接続確認が揃うまではUNKNOWNを維持し、3条件の確認後にGoalをPASSとした。managed package更新や追加のsystem-wide変更は行わなかった。

## 再利用できる知見と限界

- lingerとtimer正常だけでは、daemonのGUI非依存性を証明できない。
- live PID、親子関係、cgroup、GUI unitの`KillMode`を突き合わせる。
- oneshot launcherの状態と長寿命daemonの状態を分けて確認する。
- 起動成功ログやTCP socket数を、端末からの実接続Evidenceに置き換えない。
- GUI中、ログアウト後TTY、GUIなしbootを分けて検証する。
- 接続断でコマンドの成否が不明になったら、再実行前に実状態を確認する。

別途Hyprland/AquamarineのSIGSEGVとDRM output teardown付近のログが確認されたが、daemon修正との因果関係は確認されていない。GUIの安定性とRemote Control復旧は別課題として扱った。

これは上記環境とバージョンで検証された結果であり、他のLinux環境、将来のCodex版、長期安定性は未検証である。

公開用に個人ユーザー名、hostname、内部Driveリンク、session ID、家庭内ネットワーク情報を除去し、ホームパスを一般化した。元資料は同日付の内部Sessionと関連Codex-Worklogsである。

