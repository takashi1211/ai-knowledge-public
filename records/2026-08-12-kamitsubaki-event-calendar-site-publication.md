# 神椿イベントカレンダー（非公式）をGitHubとCloudflare Pagesで公開した記録

> このファイルはブログ記事ではなく、個人情報や秘密情報を除去した第三者向けの作業・検証記録です。

## 基本情報

- 作成日：2026-08-12
- 最終更新日：2026-08-12
- 分類：静的Webサイト制作・公開・動作検証
- 状態：一部未検証
- 元になった非公開記録：`2026-08-12_kamitsubaki-event-calendar-site-publication.md`

## 目的

すでに運用していたGoogle Calendarを第三者へ案内する入口として、非公式ファンプロジェクトであることが明確な1ページのWebサイトを作り、GitHubとCloudflare Pagesを使って公開する。PC・スマートフォンでの表示、カレンダー埋め込み、第三者によるカレンダー追加まで確認する。

## 背景

公式イベントページのRSSをAIで整理し、Google Calendarへ登録する仕組みはすでに動作していた。ただしGoogle Calendarの共有URLを直接配布するだけでは、サービスの説明や注意事項を伝えにくく、利用状況も把握しにくい。

一方、RSSに含まれない公式X、YouTube、各アーティスト個別アカウントだけの告知は自動取得できない。そこで、Webサイト上で情報源と限界を明記し、最初から多機能なポータルを目指さず、カレンダー利用に特化した入口を作った。

## 使用環境

- 機器：Windows PC
- OS：Windows（詳細不明）
- 使用ソフトウェア・サービス：HTML、CSS、JavaScript、Git、GitHub、Cloudflare Pages、Google Calendar、Cloudflare Web Analytics
- サイト構成：1ページの静的サイト

## 実施内容

### 1. 最小構成のサイト制作

`index.html`、`style.css`、`script.js` の3ファイルだけで構成した。ReactやNext.js、外部ライブラリは使用していない。

サイトには次を掲載した。

- 「神椿イベントカレンダー（非公式）」という名称
- `Unofficial Fan Project` の表記
- Google Calendarへの追加ボタン
- Google Calendarの埋め込み
- 情報源、反映範囲、AI処理の誤り、更新停止の可能性に関する注意
- ファン個人運営であり、公式とは関係がないことの明記

公式ロゴ、公式画像、キービジュアルは使わず、自作UIとテキストだけで構成した。初期版では広告、アフィリエイト、寄付等の収益化も行っていない。

### 2. PC・スマートフォン表示の修正

初回のスマートフォン表示では、タイトルが3段に分かれる、注意書きが小さい、詳細説明の段落間余白が少ないという問題があった。タイトルの折り返し、非公式表記の一体感、注意書きの可読性、段落間余白を修正し、PCとスマートフォンの両方で表示を確認した。

### 3. GitHubへの保存

Gitを導入し、静的サイトの3ファイルをGitHubのPublicリポジトリへpushした。最初のpush後にリポジトリが空の状態だったため、再度pushして解決した。

### 4. Cloudflare Pagesで公開

GitHubリポジトリをCloudflare Pagesへ接続して公開した。Cloudflareの画面では、通常の作成導線から進むとWorkers側のGitデプロイへ入ったため、Pages専用の「Looking to deploy Pages? Get started」から入り直し、「Import an existing Git repository」を使った。

公開サイト：<https://kamitsubaki-event-calendar.pages.dev>

PCとスマートフォンから外部アクセスできることを確認した。以後は、GitHubへ更新をpushするとCloudflare Pagesが自動デプロイする。

### 5. 第三者による動作確認

第三者1名にGoogle Calendar追加機能を試してもらい、「正常に追加された」と報告を受けた。制作者本人の表示確認だけでなく、第三者による実操作まで確認できた。

### 6. Web Analyticsの有効化

Cloudflare Web Analyticsを有効化し、ページへAnalytics beaconが挿入されたことをブラウザの開発者向け表示で確認した。ただし当日時点では、Web Analyticsダッシュボードへ訪問データはまだ反映されていなかった。

Cloudflareトップ画面の `Total requests` は訪問者数ではなく、HTTPリクエスト等を含む別の指標として扱う必要がある。

## 結果

- 1ページの非公式Webサイトを制作・公開できた。
- PCとスマートフォンの表示を確認できた。
- Google Calendarの埋め込みと追加ボタンを設置できた。
- GitHubからCloudflare Pagesへ自動デプロイする更新経路を構築できた。
- 第三者がGoogle Calendarへ正常に追加できた。
- Cloudflare Web Analyticsの計測コードを組み込めた。

## 失敗・注意点

- GitHubへの最初のpushは完了確認ができず、リポジトリが空だった。push後はGitHub側の内容を確認する必要がある。
- CloudflareではWorkersとPagesの導線を取り違えやすかった。Pages専用導線から作成する必要があった。
- スマートフォン表示は初回生成のままでは可読性に問題があり、実機確認後の修正が必要だった。
- RSSにない告知は、AIの精度に関係なくカレンダーへ登録できない。
- Analytics beaconの挿入と、ダッシュボードへの実データ反映は別々に確認する必要がある。

## 確認済み

- Webサイトの外部公開
- PC・スマートフォン表示
- Google Calendar埋め込み
- 第三者によるGoogle Calendar追加
- GitHubとCloudflare Pagesの連携
- Analytics beaconの挿入

## 未検証

- Cloudflare Web Analyticsへの実訪問データ反映
- 本格的な告知後の利用状況と需要
- X・YouTube等の情報を人間が共有して登録する半自動経路
- n8nセルフホストへの移行
- 将来機能の優先順位

## 得られた知見

- 小規模な情報サービスは、静的1ページ、明確な説明、追加ボタン、埋め込みだけでも公開を始められる。
- 情報収集の限界を隠さず明記することが、AIを使う非公式サービスの信頼性に重要である。
- GitHubとCloudflare Pagesの組み合わせは、静的サイトを小さく始め、更新を自動公開する構成に向いている。
- 公開後の第三者テストは、制作者本人の確認では見落としやすい利用可否を確かめるうえで有効である。
- 初期公開直後は機能を増やすより、実運用で需要と問題点を観察する方がよい。

## 関連記録

- 元のSession：`2026-08-12_kamitsubaki-event-calendar-site-publication.md`
- 関連Session：`2026-08-01_kamitsubaki-unofficial-event-calendar-planning.md`
- 関連Knowledge：現時点では未作成
- note記事：note向け完成稿作成済み
- note向け完成稿：`2026-08-12-kamitsubaki-event-calendar-from-plan-to-public-service.md`

## 公開用の処理

- 除去した情報：GitHubユーザー名、ローカルファイルパス、Calendar ID、Discord Serverの詳細、個人を特定できる情報
- 一般化した情報：使用したPC、第三者テストの依頼先
- 公開時の注意点：公開サイトURL以外のアカウント・環境情報は記載しない
- 外部公開：未実施

## 公開前チェック

- [x] パスワード、APIキー、トークン、秘密鍵を含まない
- [x] 個人名、ユーザー名、メールアドレスを含まない
- [x] IPアドレス、UUID、家庭内ネットワーク情報を含まない
- [x] 公開不要な個人的事情を含まない
- [x] 第三者の個人情報を含まない
- [x] 推測を確認済みの事実として書いていない
- [x] 未検証を明記した
- [x] 元記録との関係を記載した
- [x] 読み物へ過度に再構成せず、作業記録の役割を保った

## 変更履歴

- 2026-08-12：初版作成
