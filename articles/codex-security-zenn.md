---
title: "SASTの次は『AIセキュリティ研究者』？OpenAI Codex Securityを図解"
emoji: "🛡️"
type: "tech"
topics: ["openai", "codex", "security", "github", "typescript"]
published: true
published_at: "2026-07-30 13:21"
---

> 「脆弱性らしいコード」を見つけるだけなら、既存のSASTでもできる。  
> **Codex Securityが狙っているのは、その先だ。**

2026年3月6日、OpenAIはアプリケーションセキュリティエージェント **Codex Security** を研究プレビューとして公開しました。
さらに、CLIとTypeScript SDKを含む [`openai/codex-security`](https://github.com/openai/codex-security) も公開されています。

このツールのインパクトは、単に「AIで脆弱性を探せる」ことではありません。

**発見 → 再現 → 攻撃経路の説明 → 修正 → 再検証**までを、一つのループにしようとしている点です。

## 30秒でわかるCodex Security

```mermaid
flowchart LR
  subgraph OLD["従来のスキャナ"]
    A["ルールやパターンに一致"] --> B["警告を出す"] --> C["人間が調査・再現・修正"]
  end

  subgraph NEW["Codex Security"]
    D["コードと構成を理解"] --> E["脅威モデルを作る"] --> F["攻撃経路を探索"] --> G["隔離環境で再現"] --> H["最小パッチを提案"] --> I["人間がレビュー"]
  end
```

従来のツールは、警告を出したところで仕事が終わりがちでした。
Codex Securityは、**「本当に攻撃できるのか？」という証拠まで取りに行く**設計です。

## まず、コードではなく「システム」を見る

Codex Securityは、いきなり危険な関数を検索するのではなく、リポジトリ固有の脅威モデルを作ります。

```mermaid
flowchart TB
  TM["脅威モデル"] --> ASSET["守るべき資産・権限"]
  TM --> TRUST["信頼境界"]
  TM --> INPUT["攻撃者が操作できる入力"]
  TM --> RULE["壊してはいけない不変条件"]

  INPUT --> ENTRY["API / CLI / ファイル / Webhook"]
  ENTRY --> TRUST
  TRUST --> PRIV["権限付き処理"]
  PRIV --> DATA[("機密データ")]
```

たとえば、同じ`eval()`でも、完全に内部だけで使われる処理と、外部入力が届く処理では危険度が違います。

Codex Securityは、単語ではなく次のような流れを見ます。

```mermaid
flowchart LR
  A["攻撃者が操作できる入力"] --> B["検証不足の変換"] --> C["認証・認可境界を通過"] --> D["危険な処理"] --> E["情報漏えい・権限昇格・任意実行"]
```

## 「怪しい」を「再現できた」に変える

見つけた候補は、そのまま脆弱性として報告されるわけではありません。
検証フェーズで、到達可能性や再現結果を確認します。

```mermaid
sequenceDiagram
  participant D as Discovery
  participant V as Validator
  participant S as Sandbox
  participant P as Patcher

  D->>V: 候補 + source-to-sinkの根拠
  V->>S: 再現テスト / PoC
  S-->>V: 実行結果・証拠

  alt 到達可能で再現できる
    V->>P: validated finding
    P-->>V: 最小パッチ + 回帰テスト案
  else 根拠不足・再現不可
    V-->>D: reject または defer
  end
```

ここが重要です。

```mermaid
flowchart LR
  X["見た目が危険"] -->|従来は警告になりやすい| FP["False Positiveの可能性"]
  X -->|Codex Securityは再現を試す| PROOF["攻撃可能性の証拠"]
  PROOF --> REPORT["優先して直すFinding"]
```

## Deep Scanは「1回のAI回答」を信用しない

リポジトリには、深い探索を行うためのマルチパス・マルチエージェント構成も含まれています。
独立した探索結果を統合し、重複を除去してから、中央で検証します。

```mermaid
flowchart LR
  R["Repository"] --> W1["Discovery Worker 1"]
  R --> W2["Discovery Worker 2"]
  R --> W3["Discovery Worker 3"]

  W1 --> M["候補を統合・重複排除"]
  W2 --> M
  W3 --> M

  M --> V["中央でValidation"]
  V --> A["Attack Path Analysis"]
  A --> O["report.md / SARIF / Findings"]
```

つまり、AIの探索にばらつきがあることを前提として、**複数回の独立探索で見落としを減らす**方向です。

## 何が大きなインパクトなのか

```mermaid
flowchart TB
  I1["① セキュリティ調査のボトルネックを縮める<br/>検出後の再現・説明・修正案まで進む"]
  I2["② 開発フローへ入り込める<br/>diff / working tree / pre-commit / CI / SARIF"]
  I3["③ AppSecの判断を蓄積できる<br/>脅威モデル・ナレッジ・過去Finding・誤検知理由"]

  I1 --> I2 --> I3
```

特に大きいのは、開発者が受け取るものが「警告一覧」から、次のセットへ変わることです。

```mermaid
flowchart LR
  F["Finding"] --> POC["再現結果 / PoC"]
  F --> PATH["攻撃経路"]
  F --> SEV["実環境を考慮した深刻度"]
  F --> PATCH["最小修正案"]
  F --> TEST["回帰テスト案"]
```

OpenAIのベータ運用に関する公表値では、直近30日間に外部リポジトリの**120万超のcommit**を走査し、**792件のCritical**と**10,561件のHigh**を特定したとされています。
また、OSSへの報告では14件のCVEが割り当てられたと説明されています。

```mermaid
flowchart LR
  A["1,200,000+ commits"] --> B["792 Critical"]
  A --> C["10,561 High"]
  D["OSSへの報告"] --> E["14 CVEs assigned"]
```

:::message
これらはOpenAIが公表したベータ運用の数値であり、独立した第三者ベンチマークではありません。製品性能の保証ではなく、規模感を示す参考値として見るのが安全です。
:::

## CLIはかなり実用寄り

最小構成はシンプルです。

```bash
npm install @openai/codex-security
npx @openai/codex-security login
npx @openai/codex-security scan .
```

差分だけをCIで確認することもできます。

```bash
SCAN_ROOT="$(mktemp -d)"

npx @openai/codex-security scan . \
  --diff origin/main \
  --output-dir "$SCAN_ROOT/results" \
  --json \
  --fail-on-severity high
```

```mermaid
flowchart LR
  DEV["Pull Request / git diff"] --> CI["Codex Security"]
  CI -->|重大Findingなし| PASS["通常レビューへ"]
  CI -->|重大Findingあり| REVIEW["証拠・攻撃経路・パッチを確認"]
  REVIEW --> FIX["修正"]
  FIX --> CI
```

ほかにも、Working Treeの走査、pre-commit hook、複数リポジトリの一括走査、SARIF・CSV・JSON出力、過去スキャンとの比較、誤検知理由の記録などが用意されています。

## 「OSS化された」の意味には注意

リポジトリはApache-2.0で、CLI、TypeScript SDK、スキャン用プラグインやSkill定義を確認できます。
ただし、**完全にローカルだけで動く無料スキャナになったわけではありません**。

```mermaid
flowchart LR
  subgraph OSS["GitHubで公開されている部分"]
    CLI["CLI"] --> SDK["TypeScript SDK"]
    SDK --> PLUGIN["Plugin / Skills / Workbench"]
  end

  subgraph REQUIRED["別途必要なもの"]
    ACCESS["Codex Securityへのアクセス"]
    AUTH["ChatGPTログイン または API Key"]
    MODEL["OpenAIモデル実行"]
  end

  OSS --> REQUIRED
```

:::message
2026年7月時点のCLIは、Node.js 22.13以降の22系、24系または26系と、Python 3.10以降を必要とします。Codex Securityへのアクセスも必要です。1.0未満のため、公開APIが変更される可能性があります。
:::

## まだ「万能スキャナ」ではない

```mermaid
flowchart TB
  L["現時点の注意点"] --> L1["研究プレビューで仕様変更があり得る"]
  L --> L2["脅威モデルの前提が品質を左右する"]
  L --> L3["Deep Scanは時間・トークン・費用を使う"]
  L --> L4["生成パッチは必ず人間がレビューする"]
  L --> L5["ローカル権限・環境変数・認証情報に注意する"]
```

ローカルスキャンは、実行ユーザーのOS権限で動きます。
サブプロセスが環境変数を継承する可能性もあるため、不要なクラウド認証情報やトークンを外して実行するべきです。

結果にはソースコード断片、脆弱性の詳細、再現手順が含まれる可能性があります。
出力先はリポジトリ外に置き、閲覧権限を制限するのが安全です。

:::message alert
所有している、または明確に診断許可を得ているリポジトリだけを走査してください。生成されたパッチを自動で本番へ反映せず、通常のコードレビューとテストを通してください。
:::

そして、SAST・DAST・Dependency Scan・Fuzzingが不要になるわけでもありません。

```mermaid
flowchart LR
  SAST["SAST"] --> DEF["多層的なAppSec"]
  DAST["DAST"] --> DEF
  DEP["Dependency Scan"] --> DEF
  FUZZ["Fuzzing"] --> DEF
  CODEX["Codex Security"] --> DEF
```

## まとめ：変わるのは「検出精度」より仕事の境界

```mermaid
flowchart LR
  BEFORE["これまでの人間<br/>候補を探す・再現する・説明する・直す"] --> AFTER["これからの人間<br/>脅威モデル・重要度判断・最終レビューに集中"]
```

Codex Securityは、セキュリティ専門家を即座に置き換えるものではありません。

しかし、脆弱性診断ツールが**警告を投げるだけの存在**から、**証拠と修正案を持ってくる調査エージェント**へ変わる可能性を示しています。

「SASTキラー」と呼ぶにはまだ早いです。
ただし、AppSecのインターフェースが、静的なレポートから**反復するエージェント・ループ**へ移り始めた——そのインパクトはかなり大きいと思います。

## 参考資料

- [OpenAI: Codex Security: now in research preview](https://openai.com/index/codex-security-now-in-research-preview/)
- [GitHub: openai/codex-security](https://github.com/openai/codex-security)
- [OpenAI Help Center: Codex Security](https://help.openai.com/en/articles/20001107-codex-security)
- [Codex Security CLI quickstart](https://developers.openai.com/codex/security/cli)
