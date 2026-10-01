# ROG Ally Xで12B・14BローカルLLMとOllama／LM Studioを実機検証

> このファイルはブログ記事ではなく、個人情報や秘密情報を除去した第三者向けの実施・検証記録です。

## 基本情報

- 作成日：2026-08-01
- 最終更新日：2026-08-26
- 分類：ローカルAI / Windows / AMD iGPU / Vulkan / Ollama / LM Studio
- 状態：主要な実行結果と日常利用構成を確認済み
- 元になった非公開記録：`Sessions/2026-08-01_rog-ally-x-local-llm-ollama-lm-studio.md`

## 目的

24GB共有メモリを搭載するROG Ally Xで12B・14B級ローカルLLMを動かし、OllamaのAMD内蔵GPU利用、Thinking ON/OFF、モデル規模による速度差、LM Studioでの日常利用を実機確認する。

## 使用環境

- 機器：ROG Ally X
- メモリ：24GB（CPU・GPU共有）
- OS：Windows（詳細バージョンは記録なし）
- ソフトウェア：Ollama 0.32.5、LM Studio
- GPUバックエンド：Vulkan
- 主なモデル：Gemma 4 12B、Qwen3 14B、Gemma 4 12B QAT

## 実施内容

### 1. Ollamaの処理先を確認

初期状態のGemma 4 12Bは`ollama ps`で`100% CPU`と表示された。

Ollamaを停止し、同じPowerShellで次の環境変数を設定してサーバーを起動し直した。

```powershell
$env:OLLAMA_IGPU_ENABLE="1"
$env:OLLAMA_VULKAN="1"
ollama serve
```

別のPowerShellからモデルを実行した後、`ollama ps`で`100% GPU`を確認した。

### 2. Thinking ON/OFFを比較

Gemma 4 12Bへ同種の短い日本語回答を依頼した。

| 条件 | 総時間 | 生成トークン | 生成速度 |
|---|---:|---:|---:|
| Thinking ON | 約106秒 | 821 | 7.75 tok/s |
| Thinking OFF | 約31秒 | 245 | 8.24 tok/s |

生成速度自体の差は小さかったが、Thinking OFFでは生成量が減り、総待ち時間が大幅に短縮した。

### 3. 12Bと14Bを同一質問で比較

| モデル | Thinking | 総時間 | 生成速度 | 出力 |
|---|---:|---:|---:|---:|
| Gemma 4 12B | OFF | 約45.8秒 | 8.15 tok/s | 364 tokens |
| Qwen3 14B | OFF | 約80.8秒 | 4.76 tok/s | 380 tokens |

今回の質問ではGemma 4 12Bの方が速く、初心者向け説明としても自然だと評価した。

### 4. LM Studioへ移行

日常利用ではPowerShell操作の負担があるため、通常版LM Studioを導入した。Gemma 4 12B Instruct QAT、Q4_0、約7.15GBのモデルが画面上で`Full GPU Offload Possible`と表示された。

チャット欄の`Think`をOFFにすると、回答速度と品質のバランスが日常利用候補として良好だった。モデルをアンロードするとメモリが解放されることも確認した。

## 結果

- Ollamaの初期CPU実行を、環境変数設定と再起動によってGPU実行へ切り替えられた
- Gemma 4 12BはThinking OFFで約8.15〜8.24 tok/sだった
- Qwen3 14Bは同一質問で約4.76 tok/sだった
- Thinking OFFは生成速度よりも生成量と総待ち時間へ大きく影響した
- 通常版LM Studioと12B QATのThink OFFを、日常会話の候補として利用できた

## 失敗・訂正

- タスクマネージャーだけでは処理先を判断しにくく、`ollama ps`でCPU実行を確認した
- `Measure-Command`とOllamaのスピナー表示が干渉したため、速度測定は`--verbose`の`total duration`と`eval rate`へ変更した
- LM Studio Bionicを通常版LM Studioと誤認して導入し、用途が異なると判明後に削除した
- LM LinkをOllama連携機能と誤認したが、LM Studio同士を接続する機能だと訂正した
- 文字数指定は両モデルとも厳密には守らなかった

## 確認済み

- Gemma 4 12Bは初期状態で`100% CPU`だった
- iGPUとVulkanを有効化してOllamaを起動し直した後、`100% GPU`になった
- Thinking ON/OFFと12B・14Bの実測値を取得した
- LM Studioで12B QATをロードし、Think切り替えとアンロードを行えた
- 日常利用ではLM Studio、検証・API・自動化ではOllamaという用途分担を採用した

## 推測

- 環境変数が必要だったことは、当時のROG Ally X、Windows、Ollama 0.32.5、AMDドライバの組み合わせに依存する可能性がある
- 同じモデルでも量子化、コンテキスト長、電力設定により実測値が変わる可能性がある

## 未検証

- 環境変数をWindowsへ永続設定した場合の挙動
- 再起動後の再現性
- 長時間連続利用時の発熱、消費電力、安定性
- 中間サイズモデルとの同一条件比較
- LM StudioのAPIとOllamaを使った自動化性能の比較

## 得られた知見

- AMD共有メモリ機では、起動できるだけでなく実際の処理先をランタイム側で確認する
- Thinkingの有無はtokens/sより総待ち時間へ大きく影響する場合がある
- モデルが大きいほど常に速く高品質になるとは限らない
- 実験用ランタイムと日常利用UIを分けると運用しやすい
- 製品名や画面が想定と違う場合は、作業を進める前に公式情報で別製品ではないか確認する

## 関連記録

- 元Session：`Sessions/2026-08-01_rog-ally-x-local-llm-ollama-lm-studio.md`
- 関連Public：`Public/2026-08-12-rog-ally-x-20b-26b-local-llm-validation-public.md`

## 公開用の処理

- 除去した情報：ユーザー名、ローカルパス、個人的な会話内容
- 一般化した情報：画面操作の細部を再現に必要な範囲へ整理
- 公開時の注意点：Ollama、LM Studio、AMDドライバの現行仕様は外部公開前に再確認する
- 外部公開：未実施

## 公開前チェック

- [x] パスワード、APIキー、トークン、秘密鍵を含まない
- [x] 個人名、ユーザー名、メールアドレスを含まない
- [x] IPアドレス、UUID、家庭内ネットワーク情報を含まない
- [x] 公開不要な個人的事情を含まない
- [x] 第三者の個人情報を含まない
- [x] 訂正済みの誤認を成功例と分けた
- [x] 推測を確認済みの実測と分けた
- [x] 未検証を明記した
- [x] 元記録との関係を記載した
- [x] ブログ記事調ではなく実施・検証記録として整理した

## 変更履歴

- 2026-08-01：初版作成
- 2026-08-26：現行Publicルールに合わせ、記事形式から実施・検証記録へ再構成
