# 2GB VRAMのAlienware Alphaで4B・7B・8B級ローカルLLMを再検証

> このファイルはブログ記事ではなく、個人情報や秘密情報を除去した第三者向けの作業記録です。

## 基本情報

- 作成日：2026-08-09
- 最終更新日：2026-08-09
- 分類：ローカルAI / Linux / Ollama / GPUオフロード / 旧型PC
- 状態：一部未検証
- 元になった非公開記録：`Sessions/2026-08-09-alienware-alpha-4b-7b-8b-local-llm-revalidation.md`

## 目的

従来「1B前後が現実的」と評価していた旧型Alienware Alphaで、4B、7B、8B級モデルを実際に動かし、Ollama自動設定と手動GPUオフロードの差、日本語品質、幻覚、Markdown読解適性を確認した。

## 使用環境

- 機器：Alienware Alpha
- CPU：Intel Core i7-4770
- GPU：NVIDIA GeForce GPU（正確な製品名は未確認）
- VRAM：2048MiB
- メモリ：16GB
- OS：Bazzite
- NVIDIA Driver：580.173.02
- CUDA表示：13.0
- 使用ソフトウェア：Ollama

以前の記録では、Ollamaからcompute capability 5.0、VRAM約1.9GiBとして認識されていた。

## 実施内容

### Gemma 3 4B

自動設定：

```text
SIZE 4.8GB / 98% CPU・2% GPU / CONTEXT 4096
total duration 約14.33秒
prompt eval 16.39 tok/s
eval 6.41 tok/s
```

手動設定：

```text
FROM gemma3:4b
PARAMETER num_gpu 8
PARAMETER num_ctx 2048
```

```text
77% CPU・23% GPU / CONTEXT 2048
VRAM 1679MiB / llama-server 約1207MiB
total duration 約11.61秒
prompt eval 37.71 tok/s
eval 7.01 tok/s
```

生成速度は約9%改善し、prompt evalは大きく改善した。

### Qwen2.5 7B

自動設定：

```text
SIZE 5.2GB / 95% CPU・5% GPU / CONTEXT 4096
total duration 約21.33秒
prompt eval 21.86 tok/s
eval 3.92 tok/s
```

手動設定：

```text
FROM qwen2.5:7b
PARAMETER num_gpu 4
PARAMETER num_ctx 2048
```

```text
78% CPU・22% GPU / CONTEXT 2048
VRAM 1805MiB / llama-server 約1352MiB
total duration 約20.81秒
prompt eval 159.42 tok/s
eval 4.20 tok/s
```

`nvidia-smi dmon` では生成中のSM使用率が通常7〜9%程度、一時的に約32%まで上昇した。

### ELYZA 8B

使用モデル：`hf.co/elyza/Llama-3-ELYZA-JP-8B-GGUF:Q4_K_M`

自動設定：

```text
SIZE 5.8GB / 95% CPU・5% GPU / CONTEXT 4096
total duration 約48.94秒
prompt eval 11.78 tok/s
eval 3.69 tok/s
```

手動設定：

```text
FROM hf.co/elyza/Llama-3-ELYZA-JP-8B-GGUF:Q4_K_M
PARAMETER num_gpu 3
PARAMETER num_ctx 2048
```

```text
83% CPU・17% GPU / CONTEXT 2048
VRAM 1594MiB / llama-server 約1116MiB
total duration 約58.82秒
prompt eval 12.34 tok/s
eval 3.89 tok/s
```

約248トークンの日常会話ではtotal duration約68秒、eval約3.84 tok/sだった。

### Qwen3 4B

GPUをほぼ利用せず、Thinkingが長く、今回の環境では体感が悪かったため削除した。

## 結果

| モデル | 手動設定後の生成速度 | GPU配置 | 評価 |
|---|---:|---:|---|
| Gemma 3 4B | 約7.01 tok/s | 23% | 速度と品質の総合本命 |
| Qwen2.5 7B | 約4.20 tok/s | 22% | 長めのMarkdown読解用 |
| ELYZA 8B | 約3.89 tok/s | 17% | 日本語会話の自然さ比較用 |

実測後の評価は次のとおり。

> Alienware Alphaでは4Bは十分実用的。7Bも用途次第で実用可能。8Bも遅いが日本語会話は成立する。

## 失敗・注意点

- Gemma 3 4B：北海道を「日本最大の州」と表現し、「シベリア鉄道」を混入した。
- Qwen2.5 7B：北海道を日本最大の島と説明し、アワビ養殖、サフォーク、「シロップ用のシロギス」など不自然・架空の内容を生成した。
- ELYZA 8B：北海道の「首都」は札幌、「前進座」と呼ばれる移住者が入植した、など重大な幻覚を生成した。
- 自然な日本語と事実精度は別であり、一般知識を無条件に信用しない。
- 再生成で品質が良くなっても、GPUオフロードによる品質改善とは断定しない。

## 確認済み

- 4B、7B、8B級モデルはいずれも起動し、日本語生成を完了した。
- 手動設定でGPU配置割合を2〜5%から17〜23%へ増やせた。
- `ollama ps` のCPU/GPU比は瞬間的使用率ではなく、モデル配置割合である。
- Gemma 3 4Bは約7 tok/s、Qwen2.5 7Bは約4.2 tok/s、ELYZA 8Bは約3.9 tok/sで生成した。
- Qwen2.5 7Bは、与えたMarkdownの流れを一般知識の自由回答より正確に保持した。

## 推測

- 今回の元資料から、新たに確認済みと扱える推測はない。

## 未検証

- 長時間稼働時の安定性、温度、消費電力
- 他サービスやゲームとの同時利用
- 同一プロンプトを複数回実行した品質の統計比較
- 量子化方式の違い
- PrivateMemoryとRAGの実装

## 得られた知見

- Ollamaの初期配置だけで「GPUが使えない」と判断しない。
- 2GB VRAMでも `num_ctx` を下げ、`num_gpu` を段階的に調整すると、4B以上をCPU＋GPU併用で運用できる場合がある。
- 生成速度だけでなく、prompt eval、VRAM、実際の用途をまとめて比較する。
- 一般知識の自由回答と、与えた文書の読解は別々に評価する。
- ローカルAIはクラウドAIの完全代替ではなく、外部へ出したくない文書を扱う専用環境としても価値がある。

## 関連記録

- 元のSession：`Sessions/2026-08-09-alienware-alpha-4b-7b-8b-local-llm-revalidation.md`
- 関連Knowledge：`Knowledge/2026-08-01_alienware-alpha-small-local-llm-comparison.md`
- ブログ原稿：`Articles/2026-08-09-alienware-alpha-local-llm-revalidation-article.md`

## 公開用の処理

- 除去した情報：個人的な相談内容の具体的な文面
- 一般化した情報：PrivateMemoryで扱うセンシティブ情報の詳細
- 公開時の注意点：GPUの正確な製品名とOllamaバージョンは未確認。外部公開前に追記できると再現性が上がる。

## 公開前チェック

- [x] パスワード、APIキー、トークン、秘密鍵を含まない
- [x] 個人名、ユーザー名、メールアドレスを含まない
- [x] IPアドレス、UUID、家庭内ネットワーク情報を含まない
- [x] 健康、家庭、仕事上の個人的事情を含まない
- [x] 第三者の個人情報を含まない
- [x] 推測を確認済みの事実として書いていない
- [x] 未検証を明記した
- [x] 元記録との関係を記載した

## 変更履歴

- 2026-08-09：初版作成

