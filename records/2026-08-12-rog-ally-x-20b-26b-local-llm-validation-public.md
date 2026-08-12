# ROG Ally X 24GBで20B〜26B級ローカルLLMを実機検証

> このファイルはブログ記事ではなく、個人情報や秘密情報を除去した第三者向けの作業・検証記録です。

## 基本情報

- 作成日：2026-08-12
- 最終更新日：2026-08-12
- 分類：ローカルAI / ハンドヘルドPC / AMD iGPU / NPU / 共有メモリ
- 状態：一部未検証
- 元になった非公開記録：`2026-08-12-rog-ally-x-20b-26b-local-llm-validation.md`

## 目的

24GB共有メモリを搭載するROG Ally Xで、20B〜26B級ローカルLLMをどこまで実用速度で動かせるか確認する。

## 背景

強力な専用GPUや大容量VRAMを前提とした情報が多い一方、CPUとiGPUが同じメモリを使うハンドヘルドPCでの実測は少ない。そこで、NPUとiGPUの両経路を試し、ロード可否、GPU配置、生成速度、会話品質、失敗時のエラーを比較した。

## 使用環境

- 機器：ROG Ally X
- CPU：Ryzen AI Z2 Extreme
- GPU：Radeon 890M Graphics
- NPU：XDNA2
- メモリ：24GB共有メモリ
- OS：Windows 11
- 使用ソフトウェア：Ollama 0.32.5、FastFlowLM

## 結果一覧

| 経路・モデル | モデル配置サイズ | 配置 | context | 結果 | 生成速度 |
|---|---:|---:|---:|---|---:|
| FastFlowLM / GPT-OSS 20B | 本体約13.8GB | NPUロード試行 | 不明 | ロード失敗 | 測定不可 |
| Ollama / GPT-OSS 20B | 約12〜13GB | 100% GPU、再起動時90% GPU / 10% CPUの例あり | 32768 | 成功 | 20.17 tok/s |
| Ollama / LFM2 24B | 約15GB | 88% GPU / 12% CPU | 32768 | 成功 | 24.52 tok/s |
| Ollama / Gemma 4 26B | 約17GB | ロード未完了 | 通常 / 4096 | 失敗 | 測定不可 |

## 実施内容

### FastFlowLMでGPT-OSS 20BをNPUへロード

```powershell
flm run gpt-oss:20b
```

約13.8GB、4ファイルのダウンロード、ハッシュ検証、モデル認識までは成功したが、ロード時に失敗した。

```text
Failed to submit command to hw queue (0xc01e0200)
```

ブラウザ等を終了しても再現した。演算性能ではなくNPU用メモリ割り当てが先に限界となった可能性が高いが、原因の断定はしていない。

### OllamaのiGPU除外を解除

OllamaのデバッグログではRadeon 890MをROCm / Vulkanの両方で認識していたが、次が表示された。

```text
dropping integrated GPU; to enable, set OLLAMA_IGPU_ENABLE=1
```

次を設定してOllamaを再起動した。

```text
OLLAMA_IGPU_ENABLE=1
OLLAMA_VULKAN=1
```

### GPT-OSS 20B

`ollama ps`で約12〜13GB、100% GPU、context 32768を確認した。別の再起動時には90% GPU / 10% CPUとなる例もあった。

```text
total duration：約15.0秒
prompt eval count：75 tokens
prompt eval rate：39.30 tok/s
eval count：250 tokens
eval rate：20.17 tok/s
```

自然な日本語、複数ターンの文脈保持、構造化された相談回答を確認した。一方、自分をChatGPT / GPT-4と説明する自己認識の幻覚や、条件確認なしのGRUB修復、デバイス名の決め打ち等、危険になり得る技術回答も確認した。

### LFM2 24B

`ollama ps`で約15GB、88% GPU / 12% CPU、context 32768を確認した。

```text
total duration：約5.89秒
prompt eval count：17 tokens
prompt eval rate：16.73 tok/s
eval count：117 tokens
eval rate：24.52 tok/s
```

日本語は自然で短い雑談のテンポがよく、自分をLiquid AIのモデルと正しく説明した。一方、Linux / GRUB / apt等の技術相談では明確な誤りを確認した。

### Gemma 4 26B

約17GB。通常実行では約1GBの追加Vulkanバッファを確保できなかった。

```text
failed to allocate Vulkan0 buffer of size 1072461312
```

contextを4096へ下げたカスタムモデルも実行できなかった。

```text
ROCm error: unspecified launch failure
exit status 0xc0000409
```

## 確認済み

- FastFlowLM版GPT-OSS 20BはNPUロードで失敗した。
- Ollama版GPT-OSS 20BはRadeon 890Mへ100% GPU配置でき、20.17 tok/sで生成した。
- LFM2 24Bは88% GPU / 12% CPU配置で24.52 tok/sで生成した。
- Gemma 4 26Bは通常設定と4K contextの両方で起動できなかった。
- 20B〜24B級でも技術的幻覚は残った。

## ユーザーの所感

- GPT-OSS 20Bは、これまで使ったローカルモデルの中で圧倒的に賢そうに感じた。
- 20B〜24B級では、約1年前の無料版GPTやGeminiと会話している感覚に近い。
- 軽い雑談はLFM2 24B、少し考えさせる相談はGPT-OSS 20Bという使い分けが候補。
- 「ローカルだから我慢して使うAI」ではなく「普通に選択肢になるAI」のラインへ入った。

## 推測

- 約13〜15GBのモデル配置サイズが、この24GB環境における現時点の実用的なスイートスポット。
- 約17GBではWindows、Ollama、KV cache、Vulkan / ROCm作業領域等との競合で起動が難しくなる。
- LFM2 24BがGPT-OSS 20Bより速かったのは、MoE構造やアーキテクチャの違いが関係する可能性がある。

## 得られた知見

上限はパラメータ数だけでは決まらない。量子化、モデル構造、context、追加ワーク領域を含む実際のメモリ使用量が重要だった。

また、24GB共有メモリにより、iGPUでも13GB級モデルをGPU側へ大きく配置できた。同程度の演算性能でも専用VRAMが4〜6GBの構成では、同じ結果にならない可能性が高い。

今回の構成では、20B級はNPUよりiGPU経路が実用的だった。ただし、これは特定機器・モデル・ランタイムの結果であり、NPU一般の性能評価ではない。

## 失敗・注意点

- 自然で構造化された回答でも、危険な技術的幻覚を含む。
- GRUB、chroot、パーティション、デバイス名、管理者権限を伴う手順を無確認で実行しない。
- `ollama ps`のCPU/GPU比を瞬間的な使用率と混同しない。
- 今回の速度は単発実行であり、統計的ベンチマークではない。

## 未検証

- 長時間利用時の温度、消費電力、安定性
- 複数回測定、context統一、量子化違い
- Markdown読解、コード支援の実務比較
- Web検索、RAG、軽量CLIフロントエンド
- 30B以上、CPU / GPU / NPU協調推論

## 関連記録

- 元のSession：`2026-08-12-rog-ally-x-20b-26b-local-llm-validation.md`
- 関連Knowledge：`2026-08-12-rog-ally-x-20b-26b-local-llm-practical-limits.md`
- 事前候補Session：`2026-08-09-alienware-alpha-lan-ai-server-and-rog-ally-x-20b-candidate.md`
- note記事：`2026-08-12-rog-ally-x-20b-24b-local-llm-article.md`

## 公開用の処理

- 除去した情報：公開不要な個人的相談内容
- 一般化した情報：家庭内構成
- 外部公開：未実施

## 公開前チェック

- [x] パスワード、APIキー、トークン、秘密鍵を含まない
- [x] 個人名、ユーザー名、メールアドレスを含まない
- [x] IPアドレス、UUID、家庭内ネットワーク情報を含まない
- [x] 推測を確認済みとして書いていない
- [x] 未検証を明記した
- [x] 元記録との関係を記載した
- [x] 作業・検証記録の役割を保った

## 変更履歴

- 2026-08-12：初版作成
