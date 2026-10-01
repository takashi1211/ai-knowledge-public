# Arch Linux + Hyprlandで物理画面だけ表示不能になった際のSSH経由復旧記録

## この記録について

Arch LinuxとHyprlandを使用するハイブリッドGPU環境で、OSやデスクトップセッションは動作しているのに物理画面だけが復帰しなくなった事例を記録する。

SSH経由で行った切り分け、安全な復旧判断、`systemd-logind`を扱う際の注意点、HDMI後挿し時の挙動をまとめた。

## 使用環境

- Lenovo Legion Y720
- Arch Linux
- Hyprland
- SDDM
- Intel内蔵GPU + NVIDIA外部GPU
- 内蔵eDPディスプレイ + NVIDIA側HDMI出力
- Caelestia / Quickshell
- Aquamarine

これは特定の実機構成で発生した事例であり、すべてのHyprland環境に共通する原因を確定したものではない。

## 発生した症状

作業中、突然GUIへ戻れなくなった。

一方、SSH接続は可能で、確認時点では次のプロセスやサービスが生存していた。

- SDDM
- Hyprland
- Caelestia
- Quickshell
- 壁紙デーモン

SDDMログには、greeter sessionのクラッシュと`sddm-helper exited with 9`が記録されていた。その後、greeter自体は別のVTで再起動していた。

## logindとVTの状態

`loginctl`では、HyprlandのWayland sessionが`seat0`と`tty1`に存在し、`Active=yes`、`State=active`だった。

SDDM greeterは別のTTYに存在し、online状態だった。

物理キーボードによるVT切替を試したが、画面は変化しなかった。SSHからHyprland sessionを明示的にactivateしても、logind上はactiveになるだけで物理表示は戻らなかった。

このため、SDDMが単純にseatを占有し続けているだけでは説明できない状態と判断した。

## Hyprland側の確認

HyprlandのIPC socketは応答していた。

`hyprctl monitors all`では、内蔵eDPと外部HDMIの両方が次のように正常に見えていた。

- monitorは無効化されていない
- DPMSはON
- workspaceが割り当てられている
- 内蔵ディスプレイはfocused
- 設定エラーなし

つまり、Hyprland自身はモニターを有効なものとして認識していたが、実際の物理画面には何も表示されなかった。

## 段階的に試した復旧操作

### DPMSの再投入

SSHからDPMSを一度OFFにし、短時間待ってからONへ戻した。

物理表示は復帰しなかったため、単純な画面消灯やDPMS状態の不整合とは考えにくかった。

### Hyprlandの正常終了

既存のログアウト経路と同じ方法で、Hyprlandへ正常終了を要求した。

Hyprland終了後もSDDM画面は表示されなかった。

### HDMIの物理取り外し

この環境では、HDMI接続中のログインやセッション切替時に、Intel / NVIDIA / Aquamarine周辺が不安定になる既知傾向があった。

そこでHDMIケーブルを物理的に取り外したが、内蔵eDPにもSDDMは表示されなかった。

### SDDMの再起動

Hyprland終了およびHDMI取り外し後にSDDMを再起動した。

それでも物理表示は戻らず、単純なdisplay managerまたはgreeterの再起動では復旧できないと判断した。

## Evidenceを保存してrebootへ切り替え

SSH経由で、次の情報を保存した。

- SDDMとXorgの状態
- VTとloginctl session
- DRM connector
- Intel / NVIDIA DRM
- journal
- coredump
- greeter crash
- 障害直前の操作

OS自体は応答していたが、安全に試せるGUI側の復旧手段を使い切ったため、追加の設定変更や無制限な再起動試行は行わなかった。

必要なEvidenceを確保してから通常rebootを実施したところ、次が正常復旧した。

- SDDM
- 通常ログイン
- Hyprland
- Caelestia
- 内蔵ディスプレイ
- 既存設定

この結果から、永続的な設定破損ではなく、VT、seat、DRM、display stackのいずれかが一時的な不整合状態に入った可能性が高いと判断した。

ただし、単一の根本原因や直接のトリガーは確定していない。

## systemd-logindをGUIセッション中に再起動しない

今回の調査と構成上のリスクから、次を運用原則とした。

> GUIセッション中に`systemd-logind`を安易に再起動しない。

`systemd-logind`は次の要素と密接に関係する。

- seat
- VT
- user session
- graphical login
- display manager
- DRM device access

特に、ハイブリッドGPU、複数ディスプレイ、Wayland compositorを組み合わせた環境では、live restartによって表示やデバイスアクセスが不整合になる可能性を考慮する必要がある。

`logind.conf`またはdrop-inを変更する場合は、次の手順を採用する。

1. 設定ファイルだけを配置する
2. GUIセッション中に`systemd-logind`を再起動しない
3. 次回の通常rebootで設定を反映する
4. reboot後に実際の挙動を確認する

設定ファイルの配置と、その場でサービスを再起動して反映することは、別のリスクとして扱う。

## hypridleとlogindの役割分担

画面消灯などのユーザー空間のidle動作は、hypridle側で管理する。

今回確認した構成は次のとおり。

- 30分アイドル後にDPMS OFF
- 操作復帰時にDPMS ON
- 自動lockなし
- 自動suspendなし
- 自動hibernateなし

この構成では、画面だけが消灯し、PC本体は起動し続ける。ネットワークが正常であれば、SSHやリモート作業用サービスも継続できる。

logind側は、電源ボタンなどのsystem-level actionだけを担当させる。

## HDMI後挿し時の注意点

再起動時は安全のためHDMIを取り外していた。

ログイン後にHDMIを接続すると、外部モニターは「信号なし」となった。この環境の安定化処理が、次の起動順を前提としていたためである。

```text
ログイン前からHDMIが接続済み
  ↓
Hyprland起動時はHDMIを一時的に無効化
  ↓
一定時間後にスクリプトでHDMIを有効化
```

HDMI未接続でログインすると、遅延有効化処理が空振りする。後からHDMIを接続しても、処理は自動的に再実行されない。

既存のHDMI有効化処理を現在のセッションで1回だけ再実行すると、外部モニターは正常に復帰した。永続設定の変更は必要なかった。

このため、確認できた挙動は次のように整理できる。

- HDMIの後挿し自体は可能
- 現行構成では有効化スクリプトの再実行が必要
- 将来はhotplugイベントから同じ処理を呼び出す余地がある
- hotplug自動化は今回実装していない

## 確認済み・推測・未検証

### 確認済み

- GUI表示不能中もSSHとOSは応答していた
- Hyprland sessionはlogind上でactiveだった
- Hyprlandは両モニターを有効、DPMS ONとして認識していた
- VT切替、session activate、DPMS再投入、Hyprland正常終了、HDMI取り外し、SDDM再起動では復旧しなかった
- 通常reboot後に表示環境は正常復旧した
- HDMI後挿し後、既存有効化処理の再実行で外部モニターが復帰した

### 推測

- VT、seat、DRM、display stackの一時的不整合だった可能性が高い
- ハイブリッドGPU構成では、GUIセッション中のlogind再起動が同種の問題を引き起こすリスクがある

### 未検証

- 障害の単一の根本原因
- 同じ状態を再現できるか
- logindのlive restartが同じ障害を確実に起こすか
- HDMI hotplug処理の自動化

危険を伴う再現テストは行っていない。

## 得られた教訓

1. GUIが表示されなくてもSSHが生きていれば、再起動前に詳細なEvidenceを保存できる。
2. compositor、session、monitor、DPMSが正常に見えても、物理表示だけ失われる状態はあり得る。
3. DPMS、VT、compositor、display managerの個別操作で戻らなければ、DRM / VT / seat全体の不整合を考慮する。
4. 安全な調査範囲を使い切った後は、Evidenceを保存して通常rebootへ切り替えることが合理的な場合がある。
5. 完全な原因解明を復旧の必須条件にしない。
6. GUIセッション中の`systemd-logind`再起動には慎重になる。
7. idle時の画面消灯とsystem-levelな電源操作は、hypridleとlogindに分けて管理する。
8. HDMI遅延有効化処理は、ログイン時にHDMIが接続されているという前提を持つ場合がある。

## 公開用の処理

- ユーザー名、ローカルIPアドレス、個人用保存パスを除去または一般化した
- SSHの認証情報や秘密鍵は含めていない
- ローカル環境固有のEvidence保存場所は公開していない
- 根本原因を確定事項として記述していない
- 実機で確認した内容と推測・未検証事項を分離した

## 変更履歴

- 2026-09-14：初版作成
