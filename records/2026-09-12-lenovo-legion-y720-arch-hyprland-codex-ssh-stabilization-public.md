# Lenovo Legion Y720への外付けArch Linux導入とCodex/SSH中心のHyprland安定化

> これは、Lenovo Legion Y720へ外付けNVMe SSDからArch Linuxを導入し、Codex CLIとSSHを活用してHyprland環境を構築・安定化した実施記録です。特定ユーザーの認証情報、家庭内ネットワーク情報、ローカルパス、SSDのシリアル番号などは除去しています。

## 概要

旧世代ゲーミングノートのLenovo Legion Y720に、外付けNVMe SSDを起動ディスクとしてArch Linuxを導入した。

最初は最小構成からGUI環境を構築する必要があったが、初期段階でOpenSSH、Codex CLI、Git、base-develを導入したことで、別のPCからSSH経由で実機に接続し、Codex CLIへ調査・設定変更・ログ確認・原因切り分けを委譲できた。

最終的には、Hyprland、Waybar、Kitty、Wofi、Fcitx5、PipeWire、SDDM、NVIDIAドライバなどを組み合わせ、内蔵ディスプレイと外部HDMIモニターによる2画面環境を安定動作させた。

## 使用環境

- 機器：Lenovo Legion Y720
- 内蔵GPU：Intel HD Graphics 630
- 外付けGPU：NVIDIA GeForce GTX 1060 Mobile
- 内蔵ディスプレイ：eDP接続、1920×1080
- 外部ディスプレイ：HDMI接続、2560×1440
- 起動ディスク：外付けNVMe SSD、256GB
- OS：Arch Linux
- デスクトップ：Hyprland
- ログインマネージャー：SDDM
- 日本語入力：Fcitx5 + Mozc

## SSDの確認と交換

以前使用していた約500GBのNVMe SSDを外付けケースで再利用する予定だった。しかし、複数の環境で正常に認識できなかった。

- Windowsのディスク管理では「不明 / 初期化されていません」
- CrystalDiskInfoでは認識されない
- 初期化時に「ファンクションが間違っています」と表示される
- 別のゲーム用携帯PCでも認識されない
- Arch LinuxのLive環境でもブロックデバイスとして認識されない

複数のOS・機器・確認方法で同様の結果になったため、外付けケースではなくSSD本体の故障可能性が高いと判断した。

新たに中古のSamsung NVMe SSDを用意したところ、WindowsとCrystalDiskInfoで正常に認識された。これにより、外付けNVMeケース自体は正常であることも確認できた。

## Arch Linuxの導入

Arch Linux公式ISOをRufusでUSBメモリへ書き込み、Lenovo Legion Y720のBoot MenuからEFI USB Deviceを選択して起動した。Secure BootによってUSB起動がブロックされたため、BIOSでSecure Bootを無効化した。

インストール対象ディスクは、容量だけで判断せず、型番、シリアル番号、接続方式を確認して特定した。

```text
lsblk -o NAME,SIZE,MODEL,SERIAL,TRAN,TYPE,MOUNTPOINTS
```

archinstallでは、主に次の構成を選択した。

- 日本のミラー
- 外付けNVMe SSDのみをインストール対象に指定
- GPT / EFI
- Btrfs、subvolume、zstd compression
- zram swap
- systemd-boot
- linux kernel
- NetworkManager
- Asia/Tokyo、NTP有効
- Minimal profile
- OpenSSH、Git、base-devel、Codex CLI

## SSHとCodex CLIを先に導入した効果

Arch Linuxの初回起動時点では、GUI、ウィンドウマネージャー、日本語入力、GPU設定などが未完成だった。

しかし、OpenSSHを有効化して別PCから接続できるようにしたことで、以後の作業経路を次のようにできた。

```text
別PC → SSH → Arch Linux実機 → Codex CLI
```

この構成により、実機のハードウェア構成、パッケージ、systemdサービス、GPUドライバ、Hyprland設定、クラッシュログなどの確認と修正をCodex CLIへ依頼できた。

GUIが壊れたり、ログインできなくなったりしても、SSH接続が維持できれば復旧作業を続けられた。Linux最小構成にSSHとCodex CLIを加えた環境は、GUI構築のための強い作業基盤になった。

## NVIDIAドライバとHyprland

Lenovo Legion Y720はIntel iGPUとNVIDIA dGPUを搭載するハイブリッドGPU構成で、外部HDMI出力はNVIDIA側に接続されていた。

搭載GPUがPascal世代のGTX 1060だったため、通常の現行NVIDIAドライバではなく、対応する580系Legacy proprietary driverを使用した。DKMSの状態を確認し、再起動後に`nvidia-smi`でGPUとドライバが認識されていることを確認した。Hyprlandも起動できた。

デスクトップ環境には、Hyprland、Waybar、Kitty、Wofi、mako、PipeWire、WirePlumber、NetworkManager、SDDM、swaylock、grim、slurp、xdg-desktop-portal-hyprland、Chrome、pavucontrol、Concord、btop、cavaなどを導入した。

外部のdotfilesは参考にしたが、設定全体を丸ごと上書きしなかった。Kittyの配色、WaybarのCSS、壁紙、装飾、アニメーションなど、見た目に関する部分を中心に取り入れ、GPU、モニター、入力デバイス、Fcitx5、NetworkManagerの設定は維持した。

## 日本語入力、操作UI、音声

Fcitx5とMozcを導入し、内蔵キーボードと外付けキーボードで異なる配列をデバイスごとに設定した。Ctrl+Space、Henkan、MuhenkanでMozcを切り替え、Super+SpaceはWofiランチャーに割り当てた。

主な操作はSuper+Enter（Kitty）、Super+Space（Wofi）、Super+1〜4（workspace）、Super+Shift+1〜4（移動）、Super+Q（閉じる）、Super+R（reload）、Super+Shift+E（logout）とした。Waybarにはworkspace、時計、音量、ネットワーク、バッテリー、tray、電源メニューを表示した。

GTK3/GTK4はAdwaita darkへ統一し、pavucontrolはウィンドウクラスを確認してfloating、固定サイズ、中央配置にした。PipeWireとWirePlumberは正常に動作し、USB DAC、USB Audio、内蔵Audio、NVIDIA HDMI Audioを確認した。

## ConcordとSDDM

ConcordをArch Linuxへ導入し、Kitty Graphics Protocol、アバター、画像表示を設定した。Linux上のKittyでは、Windows、Tabby、WezTermで使った場合よりも画像表示がきれいに見えた。

最初に試したsddm-astronaut-themeは表示崩れが発生したため、SilentSDDMへ移行した。Qt6関連の依存パッケージやフォントを用意し、公式リポジトリにないフォントはAURのPKGBUILDを確認して通常のビルド手順で導入した。必要なQMLパスとQt仮想キーボード環境変数を設定し、SilentSDDMから正常にHyprlandへログインできた。

## 最大の問題：ハイブリッドGPUとHDMI

内蔵ディスプレイとHDMI外部ディスプレイを同時に接続した状態で、ログインまたはセッション切り替え時にHyprlandがクラッシュする問題が発生した。HDMIケーブルを物理的に抜くとログインできたため、HDMI接続が再現条件の一つだと分かった。

クラッシュ時のバックトレースには、AquamarineのDRM出力切り替え・切断処理が含まれていた。

```text
Aquamarine::CDRMBackend::flushAsyncCommitEvents()
Aquamarine::CDRMBackend::cancelAsyncOutput()
Aquamarine::SDRMConnector::disconnect()
Aquamarine::CDRMBackend::~CDRMBackend()
```

確認したバージョンはHyprland 0.56.2、Aquamarine 0.15.0だった。

Intel iGPU、NVIDIA dGPU、内蔵eDP、NVIDIA側HDMI出力をログイン時に同時初期化する組み合わせが、クラッシュのトリガーになっていると判断した。これはログと再現条件に基づく実機上の判断であり、上流における根本修正まで確認したものではない。

## 安定化策

起動時にすべてを同時に初期化しない構成へ変更した。

```text
Hyprland起動
  ↓ 約4秒
awww-daemon起動
  ↓ 約7秒
壁紙デーモン起動
  ↓ 約10秒
HDMI-A-1を有効化
```

ログイン時は内蔵ディスプレイだけを有効にし、Hyprland起動後に`hyprctl keyword monitor`でHDMI出力を有効化した。

その結果、HDMIを接続したままでも、SDDMでログインし、内蔵ディスプレイでHyprlandを起動し、約10秒後に外部ディスプレイが点灯して2画面運用へ移行できるようになった。

壁紙管理はswaybgからawwwへ移行した。当初は壁紙ファイルが1枚しかなかったため、ランダム選択を設定しても同じ画像になった。これはランダム処理ではなく、選択対象が1枚しかなかったことが原因だった。

## Recovery PointとGit

安定動作を確認した時点で、Hyprland、Waybar、Kitty、Wofi、Fcitx5、Bash、HDMI遅延有効化スクリプト、壁紙スクリプト、SDDMなどのユーザー設定をRecovery Pointとして保存した。

設定はGitでも管理し、キャッシュ、ログ、認証情報、過去のバックアップは対象外とした。これにより、今後の変更や実験で環境が壊れても、安定状態へ戻る基準点を確保できた。

## 得られた知見

- Arch Linuxのインストール対象は、容量だけでなく型番・シリアル・接続方式で確認する必要がある
- OpenSSHとCodex CLIを早期に導入すると、GUIが未完成でも構築と復旧を継続できる
- AIには実機調査、ログ確認、設定変更、原因候補の切り分け、再検証を委譲できる
- 確認済み事実と推測・仮説は分けて記録する必要がある
- ハイブリッドGPU環境では、ログイン時の初期化順序が安定性に影響する
- 外部dotfilesは見た目を参考にし、機種依存設定を無条件に上書きしない方が安全である
- 安定状態をRecovery PointとGitで保存しておくと、次の変更を試しやすい
- 「動作する」と「毎日快適に使える」は別の評価である

## 今後の方針

次回Arch系Linuxを導入する場合は、EndeavourOSやCachyOSなど初期環境が整ったArch系ディストリビューションを使うか、今回の設定をGitから取得する自分専用Archセットアップスクリプトを作ることを検討する。

素Archを一度経験したことで、Arch系完成ディストリビューションが何を自動化しているのかを理解できた。この経験自体には価値があるが、同じ規模の構築を毎回手作業で繰り返すのは負荷が大きい。

## 公開用の処理

この記録では、ユーザー名、SSH接続先IPアドレス、ローカルファイルパス、SSDのシリアル番号、認証情報、秘密鍵、トークン、家庭内ネットワーク情報などを除去または一般化した。

機器モデル、GPU構成、ソフトウェア構成、クラッシュの再現条件、ログに現れた関数名、安定化の考え方は、第三者が同様の問題を調査する際に有用と判断して残している。

## 未検証事項

- AquamarineまたはHyprlandの上流で根本修正されたか
- 別のPascal世代NVIDIAノートでも同じクラッシュが起きるか
- NVIDIAドライバの別バージョンで同じ症状が再現するか
- HDMI遅延有効化以外の回避策が同じ効果を持つか
- 長時間稼働時の発熱、電力、スリープ復帰の安定性
- 複数壁紙を使ったディスプレイごとのランダム切り替え

## 関連記録

- 元記録：2026-09-12 Lenovo Legion Y720 Arch/Hyprland/Codex/SSH安定化Session
- 関連する過去の実施記録：Lenovo Legion Y720上のGTX 1060ローカルLLM検証
- 関連する運用経験：Codex CLIとSSHを使ったLinux環境の遠隔保守

## 変更履歴

- 2026-09-12：初版Public原稿を作成。個人固有情報を除去し、Arch Linux導入、SSH/Codex運用、HyprlandのマルチGPU/HDMI問題と安定化手順を第三者向けに整理。
