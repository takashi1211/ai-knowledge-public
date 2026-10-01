# KDE Plasma / Wayland における表示安定化の実機検証記録

> このファイルは、実機で確認した構成上の判断と結果を、再利用しやすい形に整理した第三者向け記録です。

## 対象と確認範囲

KDE Plasma / Wayland、KWin、Quickshell、Krohnkite を使用するデュアルディスプレイ環境で、次の3点を実機検証した。

1. HDMI 出力のリフレッシュレートと安定性
2. 上部 Bar の予約領域とウィンドウ配置
3. Wayland native 起動による文字描画・日本語 IME の安定化

## 1. HDMI 出力は安定性を確認した周波数へ固定する

HDMI 出力を 75 Hz で使用した際、KWin の atomic/output failure と GUI の破綻が発生した。

約 60 Hz へ固定後、Workspace の切替を含む操作を確認した。検証中は KWin、Plasmashell、Quickshell、Krohnkite が継続して動作し、次は発生しなかった。

- KWin atomic/output failure
- NVIDIA Xid
- GPU reset

高いリフレッシュレートを利用できることと、その環境で日常利用に十分な安定性があることは別問題である。特定の GPU、出力、ドライバー、コンポジタの組み合わせで不安定になる場合は、安定性を確認した周波数を運用上の前提として固定し、変更時に再検証するのが安全である。

## 2. Bar の見た目の高さではなく、実際の予約領域を整合させる

floating Bar の見た目だけを調整しても、maximized window や tiling window、ログイン時に復元される window が Bar の下へ潜り込むことがある。

この環境では、Bar の edge inset を 25 px とし、`reservedThickness` と `exclusiveZone` をともに 81 px に統一した。これにより、通常 window、maximized window、Krohnkite による tiled window が Bar と重ならないことを確認した。

また、ログイン直後の重なりは、Plasma の session restore と Bar の reservation 成立順序の競合が原因だった。session restore との起動順を調整し、復元される window も予約領域の下端から配置される状態に修正した。

Wayland の shell UI では、装飾上の高さ、exclusive zone、window manager の配置開始位置、session restore の順序を一体として検証する必要がある。

## 3. XWayland 経路を避け、Wayland native 起動へ統一する

一部のデスクトップアプリで、文字表示のノイズと日本語 IME の不安定さが発生していた。旧 XWayland 起動経路を見直し、Wayland native 起動へ変更した。

あわせて、fcitx5 と Mozc を単一の IME 構成として整理し、関連する環境変数の整合と IME の重複がないことを確認した。

変更後、文字表示と日本語入力が安定した。Wayland セッションでアプリ単位の文字描画や IME に問題が出る場合、アプリ本体・IME・フォントだけでなく、実際に XWayland 経由で起動していないかを確認対象に含める価値がある。

## 結果

上記3点の調整後、対象環境では Workspace 切替、Bar 表示、window 配置、ログイン後の復元、文字表示、日本語入力を含む通常利用経路を確認した。

この記録は特定機種向けの設定値を一般化したものではない。周波数、予約領域、起動順は、使用する GPU、ディスプレイ、KWin、shell 実装、アプリ構成に応じて実機で確認する必要がある。

## 関連記録

- 元になった非公開記録：`Sessions/2026-09-20-lenovo-legion-y720-kde-plasma-caelestia-kde-v1-completion.md`

## 公開用の処理

- 機種固有の構成全体、個人向けランチャー、運用固有の安全制約、診断ログの詳細は含めていない。
- 実機で確認した結果と、他環境へ適用する際の検証上の注意に限定した。

## 変更履歴

- 2026-09-20：初版。HDMI 60 Hz 固定、Bar reservation / session restore、Wayland native 起動による文字描画・日本語 IME 安定化の実機検証を記録。
