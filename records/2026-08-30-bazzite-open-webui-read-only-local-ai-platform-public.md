# BazziteでOpen WebUIと読み取り専用Toolを組み合わせたローカルAI基盤

> このファイルはブログ記事ではなく、個人情報・秘密情報・家庭内ネットワーク固有情報を除去した第三者向けの実施・検証記録です。

## 基本情報

- 作成日：2026-08-30
- 最終更新日：2026-08-30
- 分類：Bazzite / Open WebUI / Ollama / Podman Quadlet / Automation / OpenAPI Tool / ローカルAI
- 状態：中核経路を実装・実機確認済み。一部未検証
- 元Knowledge：`Knowledge/2026-08-30-alienware-alpha-local-ai-practical-platform.md`

## 目的

古いPC上のローカルLLMを単なるチャット用途で終わらせず、定時に仕事を受け取り、最新ローカルデータを安全に読み、人間向けに説明できる共通基盤へ発展させる。

独自ジョブ基盤を一から作るのではなく、Ollama、Open WebUIのAutomationとTools、Podman、systemd、既存の決定論的スクリプトを組み合わせた。

## 対象環境

- 古い小型ゲームPC、メモリ16GB級
- OS：Bazzite 44系
- コンテナ：rootless Podman
- LLM実行：Ollama
- UI・Automation・Tool管理：Open WebUI 0.11.1
- モデル：Gemma 4 E2B / E4B
- 同居サービス：Samba、Jellyfin、バックアップ、NAS健康監視

## 役割分担

- Ollama：LLM実行
- Gemma：要約、説明、意味判断
- Open WebUI：チャット、Automation、Toolsの司令塔
- Automation：指定時刻にAIへ仕事を渡す
- OpenAPI Tool：必要なローカル情報だけを提供
- systemd / shell：確定処理とサービス管理
- NAS：データと状態の正本
- 人間：削除、移動、重要判断の最終承認

数値判定や異常判定をLLMへ任せず、スクリプトが生成した確定情報だけをLLMに説明させる。

## Open WebUIの常駐化

手動起動していた既存Open WebUIコンテナを、rootless Podman Quadletによるuser serviceへ移行した。既存Volumeを再利用し、ユーザーとチャット履歴を保持した。旧コンテナは停止状態で残し、データベースの整合バックアップも作成した。

Bazziteはimage-based / immutable系OSであるため、不要なOS image変更やパッケージレイヤリングは行わず、既存のPodman、systemd、Python標準環境を利用した。

Quadlet生成unitに通常の`enable`が使えない環境でも、`WantedBy=default.target`から生成された起動リンクとlingerにより、自動起動する構成を確認した。物理再起動後の完全復帰は未検証である。

## loopback限定Ollamaとの接続

Ollamaはloopback限定で待ち受けており、通常のrootlessコンテナネットワークから接続できなかった。OllamaをLAN公開せず、Open WebUIコンテナをhost networkへ接続した。Open WebUI側の待受は家庭内LANの特定アドレスへ限定した。

この判断は環境依存であり、host networkを使う場合はOpen WebUI自身の待受範囲も確認する必要がある。

## Automation実証

Open WebUIのネイティブAutomationで一回限りの自動実行を作成し、予定時刻の約5秒後にworkerが取得、status=success、エラーなし、実行履歴チャットの永続化を確認した。

これにより、人が画面を操作しなくても、指定時刻にOpen WebUIからOllama / Gemmaへ仕事を渡せることが分かった。

ただしLLMへ現在日時が自動で渡るとは限らない。現在日時が必要な処理では、Toolや外部入力で明示する必要がある。

## 読み取り専用OpenAPI Tool

最初の実用品として、既存NAS健康監視が生成する短いSTATUSファイルだけを読む専用OpenAPIサーバーを追加した。

提供するAPIは`GET /status`だけで、次は実装しなかった。

- 任意パス
- POST / PUT / PATCH / DELETE
- shell実行
- ファイル書き込み
- NAS全体へのアクセス

安全策：

- localhostのみで待受
- STATUSをread-only bind mount
- シンボリックリンク拒否
- 通常ファイル、UTF-8、最大128KiBに限定
- キャッシュ禁止
- systemd hardeningを適用
- LANからToolポートへ直接接続できないことを確認
- API取得内容と原本のSHA-256一致を確認

Open WebUIから実際にToolが呼ばれ、GemmaがNASの総合状態、SMART、温度、容量、ファイル共有、バックアップ、警告、更新時刻を日本語で説明し、ToolのSource表示も付いた。

## Native Function Callingの問題

Gemma 4 E4BでNative Function Callingを使うと、Toolを選ばない、権限がないと回答する、応答が完了しない等の不安定さがあった。OpenAPI definition取得は正常だった。

対象チャットだけLegacy Function Callingへ変更すると、GET、Source表示、内容取得、要約まで成功した。

「tool calling対応」というモデル表記だけでは、特定ランタイムとUIでNative動作が安定する保証にはならない。実際のFunction Calling方式まで含めて検証する必要がある。

## E2B / E4Bの使い分け候補

短く構造化されたSTATUS説明では、E2Bも新規チャット・過去履歴非依存で主要項目を正しく説明した。完了は約52秒だった。E4Bでは同様の処理に約2分かかった例がある。

限定テストのため一般性能差とは断定しないが、次の分担が候補になった。

- E2B：短いSTATUS、簡単な要約、単純分類
- E4B：複数資料比較、複雑な意味判断、長い文章化

## 省資源運用

モデルアンロード後、Open WebUIは約124MB、Jellyfinは約184MBで稼働し、システムのavailable RAMは約11GiBだった。E2Bロード中はモデル約7.1GB、CPU Package約62°C、アンロード後は約53°Cへ戻った。

この限定測定では、UIとToolを軽量常駐させ、モデルは必要時だけロードし、処理後アンロードする構成が成立した。電力と長期間安定性は未検証である。

## 確認済み

- 既存Open WebUIデータを保持したQuadlet移行
- Open WebUIからOllama / Gemmaの日本語チャット
- Automationの予定実行と履歴永続化
- 読み取り専用Tool経由の最新STATUS取得
- Tool取得内容と原本の一致
- 書き込み系APIの不在・拒否
- E2B / E4BによるSTATUS説明
- Legacy Function Callingによる互換回避
- モデルアンロード後の軽量常駐
- 既存Samba、Jellyfin、監視、バックアップ等の継続稼働

## 未検証

- 物理ホスト再起動後の完全復帰
- STATUSが別inodeへ原子的置換された場合の追従
- 長期間のtool calling安定性
- Native Function Callingの将来改善
- AutomationとToolの組み合わせ
- RAG、AI-Knowledge検索、Web検索
- 通知、音声、クラウドへの生成物保存
- 電力、長期間運用、他サービス高負荷時の競合

## 再利用できる判断

1. 独自ジョブ基盤を作る前に既存UIのAutomationとTool機能を確認する。
2. immutable系OSではコンテナとsystemdを優先し、ホスト変更を最小化する。
3. 判定は決定論的処理、説明はLLMへ分ける。
4. NAS全体をAIへ見せず、用途ごとの専用読み取りToolを作る。
5. Toolは読み取り専用、固定入力、小さい出力から始める。
6. 常駐コストと推論時コストを分けて測る。
7. 小さい仕事には小さいモデルを使う。
8. tool calling対応表記を鵜呑みにせず実機で確認する。

## 関連記録

- 元Knowledge：`Knowledge/2026-08-30-alienware-alpha-local-ai-practical-platform.md`
- 関連Knowledge：`Knowledge/2026-08-16-local-ai-practical-environment-architecture.md`
- 関連Knowledge：`Knowledge/2026-08-22-local-ai-resource-efficient-on-demand-architecture.md`
- 関連Knowledge：`Knowledge/2026-08-23-alienware-alpha-nas-health-monitoring-recovery.md`
- 関連Public：`Public/2026-08-23-bazzite-nas-health-monitoring-recovery-public.md`

## 公開用の処理

- 除去：家庭内IP、ユーザー名、一時SSH鍵に関する識別情報、内部識別子
- 一般化：ホームディレクトリ、service名の一部、家庭内LANの待受先
- 維持：localhost、一般的なポート・API設計、実測値、ソフトウェア名、確認済み・未検証の区別
- 外部公開：未実施

## 公開前チェック

- [x] パスワード、APIキー、トークン、秘密鍵、公開鍵本文を含まない
- [x] 個人名、ユーザー名、メールアドレスを含まない
- [x] 家庭内IP、UUID、固有デバイス識別子を含まない
- [x] 認証情報を含まない
- [x] 推測を確認済み事実として扱っていない
- [x] 未検証を明記した
- [x] 元Knowledgeとの関係を記載した
- [x] ブログ記事へ過度に再構成していない

## 変更履歴

- 2026-08-30：初版作成
