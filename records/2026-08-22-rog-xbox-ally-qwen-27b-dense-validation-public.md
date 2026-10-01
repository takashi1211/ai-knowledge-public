# 24GB共有メモリ機でQwen 27B Denseを試した実機検証

> このファイルはブログ記事ではなく、個人情報や秘密情報を除去した第三者向けの作業・検証記録です。

## 基本情報

- 作成日：2026-08-22
- 最終更新日：2026-08-26
- タイトル：24GB共有メモリ機でQwen 27B Denseを試した実機検証
- 分類：ローカルAI / Windows / AMD iGPU / Vulkan / llama.cpp / Ollama / Qwen / 量子化
- 状態：確認済み
- 元になった非公開記録：`Sessions/2026-08-22-rog-xbox-ally-qwen-27b-llama-cpp-ollama-validation.md`

## 目的

24GB共有メモリを搭載するAMDハンドヘルドPCで、27B Denseモデルを実用品質の量子化に保ったままGPU常用できるか確認する。

単なる起動可否ではなく、GPU利用、生成速度、PC全体の安定性、量子化後の品質を採用条件とした。

## 背景

同じ環境では、約13GBのgpt-oss 20Bと約14〜15GBのLFM2 24BがOllama / Vulkanで実用動作していた。一方、約17GBのGemma 4 26Bは追加バッファを確保できず起動できなかった。

Qwen 27B Denseには11〜14GB級の3bit GGUFがあり、24GB共有メモリ機でも使える可能性があった。ただし2bit級は品質低下が懸念されるため、IQ3 / Q3級を常用候補の下限とした。

## 使用環境

- 機器：ROG Xbox Ally
- CPU：Ryzen Z2 Extreme
- GPU：Radeon 890M
- メモリ：24GB共有メモリ
- OS：Windows
- 使用ソフトウェア：llama.cpp、Ollama
- llama.cpp：0.1.2-dev、build 10545、commit `a30273376`
- GPUバックエンド：Vulkan
- 主なcontext条件：4096

## 実施内容

### llama.cpp / IQ3_M

Bartowski版Qwen 27B IQ3_M（約13〜14GB級）を次の条件で試した。

```text
-ngl 999
-c 4096
-np 1
--flash-attn on
```

モデルのダウンロードは完了したが、GPUロード時に失敗した。

```text
Device memory allocation ... failed
failed to allocate Vulkan0 buffer
ErrorOutOfDeviceMemory
```

`-ngl 50`等でGPUオフロード量を減らしても、大きなVulkanバッファの確保に失敗した。

### llama.cpp / CPU側ロード

`-ngl 0`ではIQ3_Mのロードに成功し、CLIの入力待ちまで到達した。

この結果から、モデルファイル自体が壊れているのではなく、GPUオフロード時のVulkanメモリ確保が主な問題と判断した。

### llama.cpp / IQ3_XXS

約12GB級のIQ3_XXSへ軽量化したが、こちらもロードに失敗した。

```text
vk::Queue::submit: ErrorOutOfDeviceMemory
```

### Ollama / 27Bモデル

Ollamaで通常取得できる約17GBの27Bモデルも試した。初回は`PROCESSOR 100% CPU`で起動した。

次の環境変数を設定し、既存Ollamaサーバーを終了してからGPU設定を反映した。

```cmd
set OLLAMA_IGPU_ENABLE=1
set OLLAMA_VULKAN=1
set HIP_VISIBLE_DEVICES=-1
set ROCR_VISIBLE_DEVICES=-1
set GGML_VK_VISIBLE_DEVICES=0
ollama serve
```

GPU設定反映後は、次のメモリ確保エラーで起動できなかった。

```text
Failed to allocate pinned memory
vk::Device::allocateMemory: ErrorOutOfDeviceMemory
```

## 結果

| 経路・モデル | サイズ | 条件 | 結果 |
|---|---:|---|---|
| llama.cpp / IQ3_M | 約13〜14GB | GPU全オフロード | Vulkan OOM |
| llama.cpp / IQ3_M | 約13〜14GB | GPUオフロード削減 | Vulkan OOM |
| llama.cpp / IQ3_M | 約13〜14GB | `-ngl 0` | CPU側でロード成功 |
| llama.cpp / IQ3_XXS | 約12GB | Vulkan GPU | Vulkan OOM |
| Ollama / 27B | 約17GB | CPU-only | 起動成功 |
| Ollama / 27B | 約17GB | iGPU / Vulkan | pinned memory / Vulkan OOM |

27B DenseそのものがRAMへ載らないのではなく、Windows、モデル本体、KV cache、Vulkan作業領域、pinned memory等を同時に確保する段階でGPU側の余裕が不足している可能性が高い。

## 失敗・注意点

- 既存Ollamaサーバーが起動中だと、新しい環境変数を設定したサーバーを起動できず、GPU設定が反映されない。
- モデルファイルのサイズだけでGPUロード可否を判断できない。
- CPUでロードできることと、GPUで実用速度を得られることは別である。
- IQ2系は品質を優先する方針から試していない。
- 今回の結果を別OS、別Vulkan実装、別量子化へ一般化しない。

## 確認済み

- IQ3_MはCPU側ならロードできた。
- IQ3_MとIQ3_XXSはVulkan GPUロードでOOMとなった。
- Ollamaの27BモデルはCPU-onlyなら起動した。
- OllamaでiGPU / Vulkan設定を反映するとメモリ確保に失敗した。
- llama.cppとOllamaの両方で、27B DenseはGPU常用条件を満たさなかった。

## ユーザーの所感

- 動けばよいのではなく、実際に使えなければ意味がない。
- 2bitまで量子化して起動だけを狙うより、品質と速度を両立できるモデルを使いたい。
- 限界検証ではllama.cppが便利だが、常用のモデル管理はOllamaの方が扱いやすい。
- 今後は27B Denseより、active parameterが小さいMoEモデルを優先したい。

## 推測

- 現環境の実用域は、モデル配置サイズ13〜15GB程度の20B〜24B級にある可能性が高い。
- Dense / MoE、KV cache、Vulkan作業領域、CPUオフロード量の違いが、同程度のファイルサイズでも大きな差を生む。
- LFM2 24Bが良好に動作する一方で27B Denseが失敗したことから、総パラメータ数よりアクティブパラメータ量とメモリ構成が重要と考えられる。

## 未検証

- IQ2系の起動可否と品質
- Linux / ROCm等の別環境
- KV cache量子化、MTP、speculative decoding等の追加調整
- CPU-onlyでの長時間運用速度
- Qwen 3.8 35B-A3B等の新しいMoEモデル

## 得られた知見

- 24GB共有メモリ機の現実的な常用域は、現時点では20B〜24B級だった。
- 27B DenseはCPUロードできても、Windows / Vulkanで実用品質の3bit級をGPUへ載せられなかった。
- 「モデルが起動する」と「常用できる」は分けて評価する必要がある。
- llama.cppは限界の切り分け、Ollamaは普段のモデル管理と常用に向く。
- 起動だけを目的に量子化を下げ続けないことも、実用品質を探すうえで重要な判断になる。

## 関連記録

- 元のSession：`Sessions/2026-08-22-rog-xbox-ally-qwen-27b-llama-cpp-ollama-validation.md`
- 関連Knowledge：`Knowledge/2026-08-12-rog-ally-x-20b-26b-local-llm-practical-limits.md`
- 関連Public：`Public/2026-08-12-rog-ally-x-20b-26b-local-llm-validation-public.md`
- note記事：note向け完成稿作成済み
- note向け完成稿：`Articles/2026-08-22-rog-xbox-ally-qwen-27b-practical-limit-article.md`（当時の保存先。現行AI-Knowledgeでは `Articles/` を使用しない）

## 公開用の処理

- 除去した情報：ユーザー名、ローカルファイルパス、公開不要な個人的情報
- 一般化した情報：会話中の細かなダウンロード経緯を検証に必要な範囲へ整理
- 公開時の注意点：モデル名、量子化版、Ollama / llama.cppの現行仕様は投稿時点で再確認する
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

- 2026-08-22：初版作成
- 2026-08-26：旧 `Articles/` 参照が当時の保存先であることを明記
