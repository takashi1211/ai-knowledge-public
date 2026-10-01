# Arch Linux + Hyprlandで、idle消灯・Caelestia Suspend・電源ボタンSuspendを実機検証した記録

これは、Lenovo Legion Y720のArch Linux / Hyprland / Caelestia環境で、電源管理を修正し実機検証した記録です。

## 環境

- Arch Linux
- Hyprland
- Caelestia Shell
- SDDM
- Intel HD Graphics 630 + NVIDIA GTX 1060 Mobile
- 内蔵eDPディスプレイ + HDMI外部ディスプレイ

この環境では、ログイン直後のmulti-GPU初期化を安定させるため、外部HDMIを遅延して有効化する既存構成を利用していました。今回の作業では、このGPU・HDMI起動経路を変更せず、電源管理だけを対象にしました。

## 対象にした問題

- 30分アイドル後のディスプレイ自動消灯が実運用で機能していない
- Caelestiaのメニューからスリープできない
- 物理電源ボタン短押しがShutdownになる
- 設定調査中にHDMI外部ディスプレイが一時的に無効化された

## 1. idle消灯が動かなかった原因

設定ファイルには、30分アイドル後にDPMS OFF、入力時にDPMS ONという設定が存在していました。

しかしruntimeを確認すると、`hypridle.service`は有効化済みでも、実際のHyprlandセッションでは起動していませんでした。依存する`graphical-session.target`がinactiveだったためです。

設定があることと、実際に動いていることは別でした。

起動はHyprland設定ファイルへ直接追加せず、Wayland socketとHyprland IPCが利用可能になったことを待ってからhypridleを起動するuser systemd経路へ分離しました。

これにより、GPU・monitor・HDMI初期化設定を触らずにhypridleを確実に起動できるようになりました。

### 動画再生中に消灯しない件

ChromeやYouTubeの再生中はVideo Wake Lockとidle inhibitorが有効になり、自動消灯しません。

これは異常ではなく、動画視聴中に画面を消さないための意図された挙動です。

## 2. Hyprland reloadとHDMI disable

hypridle起動を`hyprland.conf`へ追加した際、設定編集を検知したHyprlandが自動reloadしました。

既存baselineには、ログイン時の安定化用として外部HDMIを一度disableする設定がありました。そのためreload時に、すでに表示中だった外部ディスプレイにもdisableが再適用され、モニターが「信号なし」になりました。

切り分けの結果、DPMS処理やhypridleそのものではなく、設定編集に伴うHyprland reloadが直接原因でした。

既存のHDMI有効化処理を再実行すると復帰します。

そのため、電源管理のためにHyprland設定を編集する方式は採用せず、hypridleの起動を外部のuser systemd経路へ分離しました。

## 3. CaelestiaからSuspendできなかった原因

OS側ではSuspendが利用可能で、セッション権限も問題ありませんでした。

原因はCaelestiaの既存メニューがSuspendではなくHibernateを実行していたことでした。この環境にはHibernate用のswap / resume構成がなく、`CanHibernate=na`だったため、スリープとして使いたい操作が失敗していました。

Caelestiaの該当操作をlogin1 native Suspendへ置き換え、メニューからSuspendし、resume後にGUIが正常復帰することを確認しました。

## 4. 物理電源ボタン

物理電源ボタン短押しの動作を、ShutdownからSuspendへ変更しました。

systemd-logindのdrop-inで`HandlePowerKey=suspend`を設定し、GUIセッション中にlogindをrestartすることは避け、通常rebootで反映しました。

短押しでSuspendへ移行し、resume後に正常復帰することを確認しました。長押しによるハードウェア強制終了の動作は変更していません。

## 5. 最終実機テスト

通常設定の30分timeoutで物理テストを実施しました。

- hypridleがDPMS OFFを実行
- 30分無操作後、内蔵eDPと外部HDMIの両方が消灯
- 入力操作で両方が復帰
- 復帰後も両出力で`dpmsStatus=1`、`disabled=false`
- Hyprlandとhypridleはいずれもactive
- テスト中にidle inhibitorは存在しない

| 項目 | 結果 |
| --- | --- |
| CaelestiaメニューからSuspend | PASS |
| Suspendからのresume | PASS |
| 物理電源ボタン短押し | PASS |
| 30分idle後の両画面消灯 | PASS |
| 入力による両画面復帰 | PASS |
| HDMIの内部状態 | PASS |

## 現在の運用

- 30分無操作でディスプレイのみDPMS OFF
- 入力で両画面を復帰
- 自動lockなし
- 自動Suspendなし
- 自動Hibernateなし
- Caelestiaメニューから手動Suspend可能
- 物理電源ボタン短押しでもSuspend可能

画面消灯中もPC本体は稼働し続けるため、ネットワークが維持されていればSSHなどのリモート利用を継続できます。

## 注意点

### Video Wake Lock

動画再生中は、意図どおり自動消灯しません。

### Hyprland設定のreload

この構成では、`hyprland.conf`の編集、`hyprctl reload`、設定reload操作によって、既存のHDMI disable設定が再適用される場合があります。

その場合は、既存のHDMI有効化処理を再実行すれば復帰します。これは今回追加した仕組みではなく、hybrid GPU環境のログイン時安定化構成に由来する既存の運用上の注意です。

## 関連記録

- `2026-09-16-lenovo-legion-y720-hyprland-power-management-acceptance.md`
- `2026-09-14-lenovo-legion-y720-arch-hyprland-gui-recovery-power-management.md`
