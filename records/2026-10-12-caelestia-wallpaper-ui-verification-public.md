# Caelestia KDEの壁紙選択UI：末尾スペース条件と紹介動画の区別

Arch Linux / KDE Plasma Wayland上の独自改修を含むCaelestia KDEで、壁紙選択UIの呼び出しと実操作を確認した記録です。

## ランチャー内の壁紙選択

調査したQMLでは、壁紙モードへの遷移条件に
`>wallpaper `（末尾に半角スペース）が含まれていました。

ランチャーを開いて `>wallpaper` まで入力しても、
この条件を満たさない場合があります。
最初の失敗時のキー入力は記録していないため、
その失敗原因がスペース不足だったとは断定しません。

既存IPCでも表示を確認しました。

`quickshell ipc --pid <稼働中のPID> call launcher action wallpaper`

PIDは起動ごとに変わるため、調査時の値を固定して使わないでください。

その後、ユーザーが実機で画面表示、Images / Animated / Videos切替、
横スクロール、独自追加のSpan画面開閉を正常と報告しました。
個別操作ログや追加スクリーンショットはありません。
壁紙・Spanプリセットの実適用や動画再生は検証対象外です。

## 紹介動画の全画面・六角形UI

Caelestia KDEのREADME紹介動画では、約00:35にランチャー内選択、
約00:38–00:39に全画面カルーセル、
約00:40–00:43に六角形一覧が登場します。

調査したCaelestiaのローカル実装とupstreamには、
該当する六角形壁紙一覧が見つかりませんでした。
後半の画面は別プロジェクトSkwd-wallのWall / Geometric表示と
強く一致します。ただし動画にアプリ名や呼出操作がなく、
帰属は強い推定です。

紹介動画のすべての画面を、Caelestia本体の機能と見なさず、
ソース・起動経路・実画面を分けて確認することが役立ちました。

Skwd-wallの導入・連携は未実施です。
既存のPlasma壁紙管理や動的配色との互換性も未検証です。

## 参照

- https://github.com/ladybug-me/caelestia-kde
- https://github.com/user-attachments/assets/38b24e7f-fdd9-43db-872b-8c0ac23a44fd
- https://github.com/liixini/skwd-wall

2026-10-12の調査記録に基づきます。
外部公開用にhostname、個人用パス、内部記録の識別子、
実行時PIDを除去・一般化しました。
