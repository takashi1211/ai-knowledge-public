# 既存のHyprland環境を壊さずCaelestiaを交換可能なUIとして導入した記録

## この記録について

Arch LinuxとHyprlandを使用するハイブリッドGPU環境へ、既存の安定化設定を維持したままCaelestia Shellを導入した実機記録である。

テーマや設定一式を全面的に置き換えず、Caelestiaを交換可能な追加UIとして扱った。起動前の状態確認、ログアウト経路、壁紙連携、失敗した実験の撤回、未解決の基盤側問題を残した完成判断についてまとめる。

## 使用環境

- Lenovo Legion Y720
- Arch Linux
- Hyprland
- Intel内蔵GPU
- NVIDIA GTX 1060 Mobile
- 内蔵eDPディスプレイ
- NVIDIA側HDMI外部出力
- SDDM
- Aquamarine
- Caelestia Shell
- awww

これは特定の実機構成で確認した事例であり、すべてのHyprland環境に同じ設定が適用できることを保証するものではない。

## 基本方針

Caelestiaのために、すでに安定していたGPU、monitor、input、NVIDIA、HDMI、Fcitx5などの設定を置き換えなかった。

採用した構造は次のとおり。

```text
既存の安定したHyprland環境
        ↓
状態確認ゲート
        ↓
交換可能な追加UI
        ↓
Caelestia
```

Caelestiaをデスクトップ環境全体の基盤ではなく、後から有効化・無効化できるUI addonとして扱った。

## ハイブリッドGPU環境の既存対策

対象機では、Intel GPUがHyprlandのprimary renderer、NVIDIA GPUがsecondary rendererとして使用され、外部HDMI出力はNVIDIA側へ接続されていた。

ログイン直後に内蔵eDPとHDMIを同時に初期化すると不安定になりやすかったため、既存環境では処理を段階的に実行していた。

```text
Hyprland起動
  ↓ 4秒
awww起動
  ↓ 7秒
壁紙処理開始
  ↓ 10秒
HDMI有効化
```

Caelestia導入後も、この順序は変更しなかった。追加UIはHDMI有効化の成功後に起動するようにした。

## 全設定を移植しなかった理由

Caelestia側のdotfilesや推奨構成を全面的に適用すると、機種固有の安定化設定が上書きされる可能性があった。

次の設定は移植対象から除外した。

- Hyprland全体
- monitor、input、device、環境変数
- NVIDIAとHDMIの設定
- HDMI遅延有効化処理
- Fcitx5
- NetworkManager
- PipeWire
- SDDM
- Kitty
- 既存のsystemd設定
- 認証関連ファイル

見た目や追加機能を導入することと、既存の動作基盤を置き換えることを分離した。

## 交換可能なdesktop addon

Caelestiaを将来の別UIと交換できるよう、addon方式の切替構造を作った。

```text
desktop-addons/
├── none
└── caelestia
```

利用する操作は次のように分けた。

- `desktop-addon use none`：元のHyprland構成へ戻す
- `desktop-addon use caelestia`：Caelestiaを有効化する
- 復旧用操作：addonの状態を安全な基準へ戻す

この構造により、Caelestiaに問題が起きてもHyprland環境全体を再構築せず、追加UIだけを外せるようにした。

## 起動前の状態確認ゲート

Caelestiaを起動する前に、次の状態を確認するゲートを設けた。

- Hyprlandプロセスが生存している
- `hyprctl`が応答する
- Hyprlandの設定エラーがない
- 内蔵eDPが正常
- HDMIが正常

条件を満たさない場合、Hyprland自体を再起動したり設定を変更したりせず、Caelestiaだけを起動しない。

最終的な起動順は次のようになった。

```text
Hyprland
  ↓
既存の壁紙処理
  ↓
HDMI遅延有効化
  ↓
状態確認ゲート
  ↓
Caelestia起動
```

追加UIの障害を、基盤全体の障害へ拡大しないための境界として機能する。

## ログアウト経路の統一

Caelestiaのメニューからログアウトした際、SDDMへ正常復帰しない問題があった。

Caelestia既定のログアウト処理と、既存のHyprland用ショートカットが異なる終了経路を使っていたため、Caelestia側もHyprlandへ正常終了を要求する方式に統一した。

```json
{
  "session": {
    "commands": {
      "logout": ["hyprctl", "dispatch", "exit"]
    }
  }
}
```

これにより、CaelestiaのメニューからもSDDMへ戻り、再ログインできるようになった。

Aquamarine終了時のSIGSEGVは残ったが、Caelestia導入前から存在した基盤側の既知問題であり、Caelestiaが原因であることを示す証拠は確認できなかった。

## 壁紙と動的配色の連携

壁紙を描画する処理は既存のawwwへ任せ、Caelestiaは選択UIと配色生成を担当するbridge構成にした。

```text
Caelestiaの壁紙選択UI
        ↓
wallpaper bridge
        ↓
既存のawww
        ↓
内蔵画面とHDMIへ反映
        ↓
現在の壁紙情報を更新
        ↓
Caelestiaが配色を再生成
```

既存の壁紙ディレクトリを移動・複製・削除せず、そのまま利用した。

複数の壁紙を使い、次を実機で確認した。

- CaelestiaのUIから壁紙を選択できる
- 内蔵画面とHDMIの両方へ反映される
- 壁紙に合わせてCaelestiaの配色が変わる
- 切替後も安定動作する

## MPRIS、音量、追加表示

MPRIS対応プレイヤーを用意し、Caelestiaから次を操作できることを確認した。

- メディア情報の表示
- 再生と停止
- 前後のトラック
- 再生位置
- アートワーク

音量はCaelestia標準のPipeWire UIを使用し、追加の重複表示は避けた。

既存の使用量表示処理を参考に、Codexの短期・週次使用量をバー上で確認できるwidgetも追加した。認証情報は変更せず、UIやログへ表示しない構成とした。

## 採用しなかった機能

### Caelestiaのロック画面

試験中にクラッシュ表示が出たため採用しなかった。

### Caelestiaによる壁紙描画

既存のawwwを維持し、Caelestia自身の背景描画は無効のままとした。

### ChatGPT Desktopの自動常駐

自動起動、専用workspace、background launcherなどを試したが、ログイン時の安定性が下がり、通常利用も複雑になったため完成条件から外した。

最終的には通常ランチャーから必要時に手動起動する方式へ戻した。

### 複数の自動起動経路

systemd user unit、XDG autostartなどの経路も試したが、再ログイン時の安定性が不足したため採用しなかった。失敗した設定は使用中の構成と分離して隔離した。

## Cleanupと復旧可能性

失敗した起動器、試験用設定、診断ファイル、自動起動関連は、使用中の構成から分離した。

大きな一括削除は行わず、必要なものは隔離して次を維持した。

- 完成環境で使用中の設定
- 導入前のRecovery Point
- Gitの基準点
- 復旧手順
- Caelestiaを外して元へ戻す操作

新しいUIを導入する際は、導入方法だけでなく、取り外し方と復旧方法も完成条件に含めることが重要だった。

## 最終確認

実機で次を確認した。

- Hyprlandの正常起動
- IntelとNVIDIAのrenderer構成
- 内蔵eDPとHDMI
- 段階的なHDMI有効化
- 状態確認後のCaelestia起動
- Caelestiaのバー、サイドバー、ランチャー、ダッシュボード
- PipeWire音量UI
- MPRIS
- 壁紙選択
- awwwへの反映
- 動的配色
- SDDMへのログアウト
- 再ログイン
- Caelestiaを無効化して元のHyprlandへ戻す経路

## 既知のリスク

次は未解決だが、Caelestia導入前から存在し、要求した機能の利用を妨げなかった。

- Aquamarine終了時のSIGSEGV
- awww-daemon終了時のSIGABRT

これらは原因を解明できていないというだけでなく、実際の利用可否への影響を確認して判断した。

異常ログが残っていることと、導入した機能が未完成であることは同じではない。

## 得られた教訓

1. 既存の安定環境を、新しいテーマやdotfilesで全面的に置き換えない。
2. GPU、monitor、inputなどの機種依存設定は維持する。
3. 追加UIは交換可能なaddonとして扱う。
4. 基盤の正常性を確認してから追加UIを起動する。
5. 追加UIの起動失敗を理由に、基盤全体を自動再起動しない。
6. ログアウトなどの重要操作は、既存の安全な経路へ統一する。
7. 導入だけでなく、無効化・撤回・復旧経路まで準備する。
8. 失敗した実験は使用中の構成から分離する。
9. 原因不明の既存問題があっても、現在の完成条件への影響を確認して判断する。

## 確認済み・推測・未検証

### 確認済み

- Caelestiaを交換可能なaddonとして導入できた
- 状態確認ゲート後に起動できた
- 壁紙選択、awwwへの反映、動的配色が動作した
- ログアウト後にSDDMへ戻り、再ログインできた
- Caelestiaなしの構成へ戻せた
- 既存のハイブリッドGPU安定化処理を維持できた

### 推測

- HDMI初期化を遅延させることで、ログイン直後のGPU初期化集中を避けられている可能性がある
- Aquamarineの問題はmulti-GPU、DRM、connector、page-flipまたはmodeset周辺にある可能性が高い

### 未検証

- Aquamarine終了時SIGSEGVの根本原因
- HDMI遅延時間の最適値
- 別機種や別GPU構成での再現性
- 他のUI addonへの同じ方式の適用

## 公開用の処理

- ユーザー名、認証情報、ローカルIPアドレスを含めていない
- 個人用の絶対パスやバックアップ場所を一般化した
- 地域固有の天気設定を省略した
- 根本原因に関する推測を確認済み事実と分離した
- 特定機種での実機結果であることを明記した
- 導入前の内部Session名や非公開記録へのリンクを省略した

## 変更履歴

- 2026-09-14：初版原稿作成
