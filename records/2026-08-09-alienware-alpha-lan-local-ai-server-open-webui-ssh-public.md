# Alienware AlphaをOpen WebUI対応のLAN内ローカルAIサーバーとして使う検証

> このファイルはブログ記事ではなく、個人情報や秘密情報を除去した第三者向けの作業記録です。

## 基本情報

- 作成日：2026-08-09
- 最終更新日：2026-08-09
- タイトル：Alienware AlphaをOpen WebUI対応のLAN内ローカルAIサーバーとして使う検証
- 分類：ローカルAI / Linux / Ollama / Open WebUI / Podman / 旧型PC
- 状態：一部未検証
- 元になった非公開記録：`Sessions/2026-08-09-alienware-alpha-lan-ai-server-and-rog-ally-x-20b-candidate.md`

## 目的

4B〜8B級ローカルLLMを動かせた旧型Alienware AlphaへOpen WebUIを導入し、GUIからのチャット、Markdown読解、LAN内AIサーバーとしての利用可能性を確認する。

## 使用環境

- 機器：Alienware Alpha
- CPU：Intel Core i7-4770
- GPU：NVIDIA GeForce GPU（正確な製品名は未確認）
- VRAM：2048MiB
- メモリ：16GB
- OS：Bazzite
- 使用ソフトウェア：Ollama、Open WebUI、Podman
- 同居サービス：Samba、Jellyfin、バックアップ処理
- バージョン：今回の元資料では未確認

## 実施内容

### Open WebUIの導入

PodmanへOpen WebUIを導入し、ホスト側のOllamaへ接続した。Open WebUIから次の4モデルを選択できた。

- `gemma3-4b-gpu:latest`
- `qwen2.5-7b-gpu:latest`
- `elyza8b-gpu:latest`
- `hf.co/LiquidAI/LFM2.5-1.2B-Instruct-GGUF:latest`

### 組み込みツールの無効化

Gemma 3 4Bの利用時に次のエラーが発生した。

```text
does not support tools
```

各モデル設定の「組み込みツール」をOFFにすると、通常のチャットを利用できた。

### Markdownファイル読解

Gemma 3 4BへMarkdownファイルを添付したところ、Open WebUIに1件のソース取得が表示され、文書の中心的な論点を概ね正しく拾った回答を生成した。

今回の範囲では、Open WebUI＋Gemma 3 4BでローカルMarkdownをGUIから添付して読ませる用途が成立した。

ELYZA 8Bでは同じ読解がうまくいかなかった。原因は確定していない。

### GPU使用率

ELYZA 8Bの利用中、リソースモニター上でGPU演算使用率が100%に達する場面を確認した。

Ollamaが表示するCPU/GPU比はモデルの配置割合であり、瞬間的な演算使用率とは異なる。モデルの一部だけをGPUへ配置していても、処理中のGPU演算使用率が高くなる場合がある。

## 結果

Alienware Alpha上で、既存のSamba、Jellyfin、バックアップ処理とともに、OllamaとOpen WebUIを運用可能な状態になった。

これにより、機器の評価は単なる小型LLM実験機から、4B〜7B級を扱うLAN内常設ローカルAIサーバー候補へ変化した。

## 失敗・注意点

- Gemma 3 4Bは組み込みツールを有効にした状態ではエラーになった。
- ELYZA 8Bでは今回のMarkdown読解がうまくいかなかった。
- Open WebUIをサーバー機自身のChromeで操作すると、Ollama CLIより重く感じた。
- Open WebUI、ブラウザ、RAG、デスクトップ環境、既存サービスのどれが負荷の主因かは未測定。
- ELYZA 8Bの短いコンテキスト長がファイル読解へ影響した可能性はあるが、未検証。

## 確認済み

- Podman上のOpen WebUIからホスト側Ollamaへ接続できた。
- 4モデルをOpen WebUIから選択できた。
- 組み込みツールをOFFにすると通常チャットが利用できた。
- Gemma 3 4Bは添付したMarkdownを参照した回答を生成できた。
- ELYZA 8Bでは今回のファイル読解がうまくいかなかった。
- ELYZA 8Bの処理中にGPU使用率100%の場面を確認した。
- Samba、Jellyfin、バックアップ、Ollama、Open WebUIを同一マシン上で運用可能な状態になった。

## 推測

- GUIの描画を別端末へ分担すれば、サーバー側でブラウザを開くより使い勝手が改善する可能性がある。
- Open WebUIの重さには、ブラウザ、RAG、デスクトップ環境、同居サービスが影響した可能性がある。
- ELYZA 8Bのファイル読解失敗には、短いコンテキスト長が影響した可能性がある。

## 未検証

- 同一LAN内の別PCやスマートフォンからOpen WebUIへ接続できるか。
- SSH経由でサーバー上のOllama CLIを実行した場合の速度と使い勝手。
- Open WebUI利用時のCPU、RAM、GPU、ストレージ負荷の内訳。
- ELYZA 8Bのコンテキスト長を変えた場合のファイル読解。
- 長時間運用時の安定性、温度、消費電力。
- JellyfinやSambaと高負荷推論が重なった場合の影響。

## 得られた知見

- Open WebUIは、サーバー機自身で使うGUIだけでなく、LAN内AIサーバーへのWebフロントエンドとして考えると価値が大きい。
- GUI・ファイル添付を重視する場合はOpen WebUI、軽さ・速度・シェル上のファイル利用を重視する場合はSSH＋Ollama CLIという使い分けが候補になる。
- `ollama ps` の配置割合とリソースモニターのGPU演算使用率を混同しない。
- ローカルAIはクラウドAIの完全代替だけでなく、外部へ出したくないローカル文書を扱う独立環境としても価値がある。

## 関連記録

- 元のSession：`Sessions/2026-08-09-alienware-alpha-lan-ai-server-and-rog-ally-x-20b-candidate.md`
- 直前のSession：`Sessions/2026-08-09-alienware-alpha-4b-7b-8b-local-llm-revalidation.md`
- 関連Knowledge：`Knowledge/2026-08-01_alienware-alpha-small-local-llm-comparison.md`
- 関連Public：`Public/2026-08-09-alienware-alpha-4b-7b-8b-local-llm-revalidation-public.md`
- ブログ原稿：既存記事への将来追記候補

## 公開用の処理

- 除去した情報：LAN内IP、ユーザー名、個人的・センシティブな利用内容。
- 一般化した情報：家庭内の端末構成、実際に添付した非公開Markdownの名称と内容。
- 公開時の注意点：別端末接続とSSH利用は未検証。サーバーとしての長期安定性も未確認。

## 公開前チェック

- [x] パスワード、APIキー、トークン、秘密鍵を含まない
- [x] 個人名、ユーザー名、メールアドレスを含まない
- [x] IPアドレス、UUID、家庭内ネットワーク情報を含まない
- [x] 健康、家庭、仕事上の個人的事情を含まない
- [x] 第三者の個人情報を含まない
- [x] 推測を確認済みの事実として書いていない
- [x] 未検証を明記した
- [x] 危険な手順に注意を書いた
- [x] 元記録との関係を記載した

## 変更履歴

- 2026-08-09：初版作成

