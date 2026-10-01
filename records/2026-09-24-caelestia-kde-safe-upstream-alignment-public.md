# Caelestia KDEを既存KDE環境へ安全に寄せるための分離設計と実機検証

> この記録は、KDE Plasma / Wayland環境でCaelestia KDEの上流UIと操作感を段階的に取り込みつつ、既存デスクトップ環境や共有設定への影響を抑えた実施結果をまとめた第三者向け記録です。

## 対象と目的

KDE Plasma / Wayland、KWin、Quickshell、Krohnkiteを使用するデュアルディスプレイ環境で、Caelestia KDEを日常利用できる状態に整えた。

目的は、上流のインストーラーをそのまま適用することではない。見た目や日常操作は可能な限り上流に寄せながら、既存のKDE設定、別のデスクトップ環境、電源管理、アプリの通常プロファイルへ不用意に影響しない構成を作ることである。

最終的には、上流機能を広く止めた安全優先構成ではなく、UIと日常操作を取り込みつつ、共有領域やsystem-wide設定に触れる部分だけを分離・制限した構成になった。

## 基本方針

### 上流インストーラーを直接適用しない

上流のセットアップには、通常のXDG領域、KDE設定、パネル、KWin shortcut、電源管理、SDDM、system-wide機能へ影響し得る処理が含まれていた。

既存環境と共存させる場合は、インストーラーを直接適用する代わりに、変更を次のように分ける方針が有効だった。

- Caelestia / Quickshellの設定、データ、state、cacheはprivateなXDG領域に置く
- 外部アプリの起動は通常のhost XDG環境へ戻す
- Plasma、KWin、PowerDevilなどの共有設定は、必要な範囲以外を変更しない
- system-wideな機能は、必要性と安全性を個別に検証するまで採用しない
- 外観、パネル、shortcut、terminalなどは一度に変更せず、段階的に確認する

この分離により、Caelestiaの起動環境がブラウザやデスクトップクライアントへそのまま継承され、別プロファイルが起動する問題を防げた。

## 実施した構成

### 上流UIへの段階的な復帰

Plasma標準panelを外し、Caelestia単独で日常操作できる構成に移行した。Barはbottom配置とし、両ディスプレイでwindowと重ならない予約領域を設定した。

確認した主な要素は次のとおり。

- 上流に近いscale、spacing、padding、animation、transparency
- 5 WorkspaceとMeta+数字キーによる切替
- Launcher、Terminal、Screenshot、Emoji、Keybinds、Overview、Sidebar
- KrohnkiteのBTree layout
- Caelestia経由のterminal起動経路の統一
- Plasma panelを使わない状態でのwindow管理、tray、電源UI、session restore

上流と同じ値へ合わせること自体は目的ではない。既存のwindow manager、shortcut、display、電源管理と競合する場合は、利用環境に合わせて差分を残した。

## 実画面を設定値より優先する

設定上は同じalpha値やblur指定でも、実画面の印象が上流デモと大きく異なることがあった。

この環境では、KWin blurが暗いsurfaceの背後を平均化し、設定値以上に不透明で暗く見える傾向があった。Caelestia側のblur regionを使わない構成にしたところ、透明感はむしろ上流デモに近づいた。

この結果から、GUIの受入確認では次を優先する。

1. 実際のruntime状態
2. 実スクリーンショット
3. 設定値やsource上の一致

blurを外したことが直接の原因であるという説明は、比較結果に基づく推定である。環境ごとにKWin、壁紙、surface色、GPU、compositorの条件が異なるため、同じ結果になるとは限らない。

## idle描画負荷の見直し

音声ビジュアライザーは、PipeWire AudioCollectorとlibcava FFTを使うQuickshell内の実装にした。外部の`cava`プロセスは使用していない。

初期状態では、無音時にも複数ディスプレイ分の描画が短い周期で続いていた。次の変更を行った。

- 無音時に値をclearする
- 短いsettle時間の後にTimerを停止する
- QMLのdirect bindingを減らし、必要なタイミングでsamplingする
- providerを共有し、FFTを二重に実行しない
- full-screenのMultiEffect shadowを無効化する

完成後のidle測定では、Intel内蔵GPUのRender/3D使用率は約0.73%、clockは4 MHz、RC6は99%、消費電力は約0.02 Wだった。通常操作時、動画再生時、ビジュアライザーの表示時に負荷が増えることは想定どおりである。

この例では、常時動く視覚効果を「見た目の要素」としてではなく、idle時の描画ループ、binding、合成処理として切り分けることが有効だった。

## 安全化はページ単位ではなく操作単位で行う

設定UIを丸ごと無効にすると安全ではあるが、状態確認まで失われて実用性が下がる。

NetworkとBluetoothでは、情報表示と状態変更を分けた。

- adapter、接続状態、SSID、signal、IP、paired device、batteryなどはread-onlyで表示する
- 接続、切断、scan、forget、trust、adapter切替、IPv4変更などのmutationは止める
- UI、IPC、QML handlerの複数層でmutationを遮断する

この方式により、状態を確認できることと、意図しない設定変更を防ぐことを両立できた。

同じ考え方は、package操作、plugin操作、KWin共有設定、lockscreen、Power actionなどにも適用できる。安全化は「画面を閉じること」ではなく、「どの操作がどの設定領域を変更するか」を明確にすることとして扱う方が実用的である。

## 共有設定の所有者を残す

Caelestiaが表示できるからといって、すべてのデスクトップ機能をCaelestiaへ移管する必要はない。

この構成では、次のownershipを維持した。

- 電源管理：PowerDevil
- 壁紙：Plasma
- KDE / Qt全体のテーマ：既存設定
- Material You系のpalette：Caelestia内部のみ
- session restore：Plasma / KWin側の既存機構
- HDMI出力：安定性を確認した約60 Hzを維持
- custom SDDM、custom lockscreen、live preview、workspace tracker、Better Blur：未採用

壁紙はPlasmaの現在値をread-onlyで参照し、Caelestia用のpaletteだけを生成する構成にした。これにより、壁紙変更に合わせたaccentやthemeの変化を得ながら、KDE、Qt、他のデスクトップ環境へ色設定を書き戻さずに済んだ。

「上流完全一致」ではなく、各機能の所有者を明確にしたうえで、必要なUIだけを取り込むことが安定性につながった。

## 確認できた結果

実機操作、runtime確認、実スクリーンショットにより、次を確認した。

- Plasma panelを外した状態で、Caelestiaのみで日常操作できる
- bottom Barとwindow reservationが両ディスプレイで成立する
- 5 Workspace、Krohnkite、主要shortcut、terminal経路が動作する
- 外部アプリが通常のhost profileで起動する
- private paletteが壁紙変更に追従する
- 音声ビジュアライザーが表示され、無音idle時の描画負荷を抑えられる
- Network / Bluetoothの状態表示とmutation遮断が成立する
- terminalから起動したsystem monitorを終了後、専用terminalやprocessが残留しない

## 未検証・今後の再評価

- Bluetooth関連サービスを導入した場合のread-only情報表示
- Plasma更新後も一部周辺機器のbattery通知対策が必要か
- 上流由来の非致命QML warningの修正状況
- system-wide機能を将来取り込む場合の個別検証
- GPU、出力、KWin、driver、壁紙、ディスプレイ構成が異なる環境でのblurと描画負荷の再現性

## 適用時の要点

- 上流インストーラーの影響範囲を先に確認する
- private runtimeとhost applicationの実行環境を分ける
- panel削除、Bar移動、shortcut変更、terminal変更を同時に行わない
- UIの安全化はページ単位ではなくmutation単位で設計する
- 見た目の数値一致より実画面を評価する
- idle時のTimer、binding、合成処理を実測する
- 既存の電源管理、壁紙、session restore、共有設定のownershipを安易に奪わない
- display modeやKWinまわりの変更は、実際の利用条件で再検証する

## 関連記録

- 元Session：`Sessions/2026-09-24-lenovo-legion-y720-caelestia-kde-upstream-alignment-completion.md`
- 関連Public：`2026-09-20-kde-plasma-wayland-display-stabilization-public.md`  
  HDMI出力、Bar reservation、Wayland native起動による表示安定化は、こちらで個別に記録している。

## 公開用の処理

- 個人用のパス、アプリ名、周辺機器名、端末固有の運用情報は削除または一般化した。
- 個別環境でのみ有効な設定値を一般解として扱わず、実機確認が必要な条件として記載した。
- 実測結果、設計判断、未検証事項を区別した。

## 変更履歴

- 2026-09-25：初版。Caelestia KDEの上流UIを既存KDE環境へ段階的に取り込んだ実施結果、分離設計、描画最適化、read-only UI境界を記録。
