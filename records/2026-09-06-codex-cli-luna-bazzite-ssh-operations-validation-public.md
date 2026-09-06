# Codex CLI＋Lunaの軽量運用とSSH越しのBazzite保守を実地検証した記録

> このファイルは、個人情報や家庭内ネットワーク固有情報を除去した第三者向けの実機検証記録です。

## 基本情報

- 検証日：2026-09-06
- 分類：Codex CLI / GPT-5.6 Luna / Linux / SSH / Bazzite / rclone
- 状態：主要検証と障害復旧は完了、一部未検証
- 元Session：`Sessions/2026-09-06-codex-cli-luna-bazzite-ssh-operations-validation.md`

## 目的

RAM 2GBの古いMacBookを、ブラウザを開かずにクラウドAIと対話できる軽量端末として再活用できるか検証した。

Codex CLIの起動確認だけでなく、次の用途を実際に試した。

- 通常会話
- 接続済みGoogle Drive内の資料参照
- 複数資料を横断した情報整理
- SSH越しのBazziteサーバー調査
- rclone OAuth障害の原因切り分け
- 再認証とsystemdサービスの復旧
- 5時間利用枠の消費確認
- 長時間SessionにおけるContext windowとConversation recapの確認

## 使用環境

### クライアント

- 古いMacBook
- RAM：2GB
- OS：Linux Mint XFCE
- 主な用途：Codex CLI、ターミナル、SSH

別のWindows携帯型PC上のCodex CLIからも、モデル選択とSSH保守を試した。

### 接続先

- OS：Bazzite x86_64
- CPU：第4世代Core i7
- メモリ：約16GB
- 主なサービス：Samba、Jellyfin、Ollama、Open WebUI、rclone

## Codex CLIの導入

MintにはNode.jsとnpmが入っていなかったため追加した。

最初に次を実行した。

```bash
npm install -g @openai/codex
```

結果：

```text
EACCES: permission denied
```

その後、次で導入できた。

```bash
sudo npm install -g @openai/codex
```

ChatGPTアカウントによる初回認証にも成功した。今回はAPIキーによる従量課金ではなく、ChatGPTプラン側のCodex利用として動作させた。

## RAM 2GB環境での使用感

GUIブラウザを開く運用と比べてCodex CLIは軽く、RAM 2GBのMacBookでもクラウドAIとの会話が実用的に動いた。

追加で次のツールも導入した。

- `btop`
- `ranger`
- `tmux`

`tmux`でbtopとCodex CLIを左右に分割表示する使い方も試した。

古いPC側では表示と入力だけを担当し、重い推論をクラウドへ任せることで、「古いGUI PC」ではなく「CLI中心の軽量AI端末」という新しい役割を持たせられる可能性が見えてきた。

## GPT-5.6 Lunaの利用

本人の環境では、Codex CLIから `gpt-5.6-luna` を選択できた。

一方、確認したCodex Desktop版ではLunaが表示されず、より軽い選択肢はTerra系だった。

別のWindows機に入っていたCodex CLI v0.151.0でもLunaを選択できたため、少なくとも今回確認した環境では、CLI版とDesktop版にモデル提供上の違いがあった。

Desktop版でLunaが提供されていない公式理由は確認していない。

## Windows版Codex CLIの更新

インストールスクリプトをPowerShellから直接実行したところ、次のエラーになった。

```powershell
irm https://chatgpt.com/codex/install.ps1 | iex
```

```text
このオブジェクトにプロパティ 'OSArchitecture' が見つかりません。
```

この環境にはnpmも入っていなかった。

その後、Codex CLI自身の更新コマンドを使用した。

```text
codex update
```

結果：

```text
Updating Codex CLI from 0.151.0 to 0.153.4
Codex CLI 0.153.4 installed successfully.
```

この環境では、再インストール用スクリプトではなく `codex update` で正常に更新できた。

## Google Drive内の資料参照

ローカルへGoogle Driveをマウントしていない状態でも、Codex CLIから接続済みGoogle Drive内の資料を参照できた。

複数の個人記録を横断して一つのテーマを整理させたところ、資料ごとの断片を統合した回答が得られた。

小型ローカルAIとCLI上の操作感は近かったが、複数資料の統合や意味の整理では、クラウドAIとしての性能差を大きく感じた。

## Luna HighとMediumの比較

### High

資料を参照させて最近のPC・Linux利用方針を整理させた。

回答品質は十分だったが、次の表示となった。

```text
Worked for 3m 55s
```

待ち時間は非常に長く、この時点で5時間枠を1%消費した。

### Medium

別の複雑な個人テーマを複数資料から整理させた。

Mediumでも主要な論点を適切に分類でき、Highより体感上かなり速かった。利用枠の消費は1%だった。

今回の範囲では、日常用途はLuna Mediumを基本とし、Highはより深い整理が必要な場面に限定する使い方が有力だった。

## 通常会話

Luna Mediumで通常の雑談も試した。

単純な肯定だけでなく、CLIの低負荷、集中しやすさ、古いPCへの新しい役割、重い処理をクラウドへ任せる役割分担などを自然に整理した回答が得られた。

本人の体感では、資料整理だけでなく普段の会話にも十分実用的だった。

## SSH越しのBazzite調査

Windows側のCodex CLIから家庭内のBazziteサーバーへSSH接続させた。

接続後、Codex自身がOS、CPU、メモリ、uptimeなどを取得できた。

さらに「変更せずに健康診断し、異常や注意点だけを報告する」という制約で調査させた。

問題なしと確認された項目：

- NASディスク使用率：32%
- システム領域使用率：40%
- 利用可能メモリ：約10GiB
- CPU負荷：load average 0.02〜0.08
- Samba：稼働中
- Ollama：稼働中
- Open WebUI：稼働中
- Jellyfinコンテナ：healthy
- Jellyfinのポート：稼働中
- SMARTチェックとNAS監視タイマー：実行予定あり

一方、Google Drive関連の2つの処理が失敗していることを発見した。

- Google DriveからNASへのバックアップ
- NAS状態ファイルのGoogle Drive同期

共通して次のエラーが記録されていた。

```text
invalid_grant
```

## rclone OAuth障害の切り分け

変更を禁止した状態で追加調査させた。

確認された内容：

- 2つのrclone remoteが同じ `invalid_grant` で失敗
- Google Drive APIまでは到達していた
- アクセストークン取得段階で拒否されていた
- サーバー時刻はNTP同期済み
- 3回再試行しても同じエラーだった

これらから、時刻ずれや一時的な通信障害の可能性は低く、refresh tokenの失効、期限切れ、OAuth設定変更などが原因候補と判断された。

直接原因の確定までは行っていない。

## Google Driveバックアップの復旧

最初のremoteだけを対象として、次の制約を与えた。

- 対象remote以外は変更しない
- 必要な場合だけ再認証する
- Google Driveの読み取りを確認する
- バックアップサービスを実行する
- systemdの結果を確認する

再認証後、Google Driveの読み取りに成功した。

続いてバックアップサービスを実行し、次を確認した。

```text
Result=success
```

もう一方のremoteには触れていないことも確認した。

## NAS状態同期の復旧

続いて、もう一方のremoteだけを再認証した。

Google Drive側に保存されたNAS状態ファイルを読み取れることを確認した後、同期サービスを実行した。

結果：

```text
Result=success
ExecMainStatus=0
```

その他のrclone remoteやサービスは変更していない。

この結果、Codex CLI、Luna、SSHを組み合わせて、実際の家庭内サーバー障害について次の工程を完了できた。

```text
状態調査
→ 原因候補の切り分け
→ OAuth再認証
→ systemdサービス実行
→ 成功値の確認
```

## 5時間利用枠の実測

一連の作業ではLuna Mediumを中心に使用した。

途中の状態：

```text
5h limit: 96% left
```

この時点までに行った主な作業：

- Google Drive参照
- 複数資料の横断整理
- 通常会話
- SSH接続
- サーバー健康診断
- rclone障害調査
- 2つのremoteの再認証
- 2つのsystemdサービスの復旧確認

これらを行った後も96%残っており、今回の利用枠消費は合計約4%だった。

ただし、利用枠の消費は作業内容やサービスの提供条件などで変わる可能性がある。今回の4%は一回の実測値であり、固定的な消費量ではない。

## Context windowの消費

同じSessionを長時間使った結果、次の状態になった。

```text
Context window: 27% left (192K used / 258K)
```

5時間利用枠は96%残っていた一方、Context windowは73%使用していた。

今回の使い方では、時間枠より先に長いSessionのContext windowが問題になる可能性が見えた。

そのため、次の運用が有力と判断した。

```text
1 Session = 1まとまりの作業
```

作業が完了したら、新しいSessionへ切り替える。

## Conversation recapの誤り

Context windowを多く使った後、本人の操作なしに次が表示された。

```text
─ Conversation recap ─
```

会話の内容が自動的に圧縮・要約されたものと考えられる。

ただしrecapでは、NAS同期サービスについて「復旧確認は未実施」と記載された。

実際には、その時点ですでに次を確認済みだった。

```text
Result=success
ExecMainStatus=0
```

長いSessionでConversation recapが発生した場合、直近の作業結果や状態が誤って圧縮される可能性を実際に確認した。

重要な保守作業では、成功値を明示的に確認するとともに、作業完了後にSessionを区切る方が安全と考えられる。

## 確認済み

- RAM 2GBのLinux Mint環境でCodex CLIが動作した
- ChatGPTアカウントでCodex CLIを認証できた
- Codex CLIでGPT-5.6 Lunaを選択できた
- Luna Mediumで通常会話と資料参照が実用できた
- Codex CLIからSSH越しにBazziteを調査・操作できた
- rclone OAuth障害の調査から再認証、サービス復旧まで完了した
- 2つのsystemdサービスで成功値を確認した
- 一連の検証後も5時間利用枠が96%残っていた
- 長時間SessionでConversation recapが発生した
- recapに実際の最新状態と異なる記述が含まれた

## 現時点の判断

今回の用途では、Luna Mediumが次の作業に使える可能性が高い。

- 日常会話
- Drive内の資料参照
- 複数資料の軽い整理
- Linux状態確認
- SSH保守
- ログ調査
- 明確なルールと制約がある作業

Highは、Mediumより深い整理が必要な場合の候補になる。ただし今回の実測では待ち時間が長かった。

複雑な設計、本格的なソフトウェア開発、難しい原因分析、Lunaで解決できない作業では、より上位のモデルへ切り替える使い分けが考えられる。

## 未検証

- Desktop版でLunaが提供されない公式理由
- Luna Mediumを長期間使った場合の平均的な利用枠消費
- 他モデルとの同一条件による詳細比較
- Conversation recapが発生する正確な条件
- `invalid_grant` を発生させた直接原因
- RAM 2GBのMacBookから継続的にSSH保守する場合の安定性と使い勝手

## 得られた知見

- Codex CLIはコーディング以外にも、軽量クラウドAI端末として利用できる
- 低性能PCでも、表示と入力をCLIに寄せ、推論をクラウドへ任せる構成なら実用的になる可能性がある
- Luna Mediumでも、明確な制約を与えたLinux調査と復旧を完走できた
- 明文化されたルールや作業範囲は、軽量モデルへ任せられる範囲を広げる可能性がある
- 利用時間枠とContext windowは別々に管理する必要がある
- 長時間Sessionのrecapは、最新状態を正確に保持するとは限らない
- 重要な作業では、サービス名だけでなく `Result=success` や `ExecMainStatus=0` などの成功値を確認することが重要

## 関連記録

- `Sessions/2026-08-12-old-pc-linux-reuse-and-ai-distance.md`
- `Sessions/2026-08-16-alienware-alpha-google-drive-backup-automation.md`
- `Knowledge/2026-08-16-bazzite-rclone-google-drive-nas-backup.md`
- `Knowledge/2026-08-23-alienware-alpha-nas-health-monitoring-recovery.md`
- `Public/2026-08-12-old-pc-linux-reuse-and-ai-distance-public.md`
- `Public/2026-08-16-bazzite-rclone-google-drive-nas-backup.md`

## 公開用の処理

- Windowsユーザー名を含む実パスは掲載しなかった
- 家庭内IPアドレスや認証情報は掲載しなかった
- Google Drive内の個人的な資料内容は一般化した
- OAuth Client ID、Client Secret、token、rclone設定内容は掲載しなかった
- 一台または本人環境での実測であり、すべての環境で同じ結果になるとは限らない

## 公開前チェック

- [x] パスワード、APIキー、token、秘密鍵を含まない
- [x] 個人名、メールアドレス、ユーザー名を含まない
- [x] IPアドレスや家庭内ネットワーク固有情報を含まない
- [x] 確認済み、判断、未検証を区別した
- [x] 失敗した手順とrecapの誤りを残した
- [x] 元Sessionとの関係を記録した
- [x] 一回の実測値を一般的な保証として記載していない

## 変更履歴

- 2026-09-06：初版作成
