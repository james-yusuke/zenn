---
title: "OpenMythosはFable 5 / Mythos 5を再現できているのか"
emoji: "🧭"
type: "tech"
topics: ["ai", "claude", "anthropic", "llm", "pytorch"]
published: true
published_at: "2026-07-03 05:28"
---

:::message alert
この記事は2026-06-29時点の公開情報だけで検証しています。結論はシンプルです。OpenMythosは「公式再現」ではありません。公開論文を組み合わせた「仮説実装」として読むのが精確です。
:::

## 0. まず結論

```mermaid
flowchart LR
  Q["OpenMythosは<br/>本物のFable 5 / Mythos 5か？"]
  A["No<br/>公式再現とは言えない"]
  B["Yes寄り<br/>研究仮説としては筋がある"]
  C["理由1<br/>Anthropicは内部構造を非公開"]
  D["理由2<br/>OpenMythos自身が<br/>speculative reconstructionと明記"]
  E["理由3<br/>安全機構・学習済み重み・公式検証がない"]
  Q --> A
  Q --> B
  A --> C
  A --> D
  A --> E
```

**判定:** 公式モデルの正しい再現ではない。RDT・MoE・MLAなどを使った、教育的で面白い「推測アーキテクチャ」です。

---

## 1. 今回の情報源

```mermaid
flowchart TB
  S["調査対象"] --> O["Anthropic公式<br/>最優先"]
  S --> D["Claude Platform Docs<br/>仕様確認"]
  S --> G["OpenMythos GitHub<br/>コード確認"]
  S --> P["arXiv論文<br/>部品の妥当性だけ確認"]
  S --> N["Reuters等<br/>提供状況だけ確認"]
  X["使わない"] --> X1["Xの噂"]
  X --> X2["未出典ブログ"]
  X --> X3["スクショだけの投稿"]
```

ZennはMermaidを公式に表示できます。この記事の図は、すべてZenn上でそのままレンダリングされるMarkdown図です[^zenn-mermaid]。

---

## 2. Fable 5 / Mythos 5で公式に言えること

```mermaid
flowchart LR
  M["同じ基盤モデル"] --> F["Fable 5"]
  M --> Y["Mythos 5"]
  F --> F1["一般利用向け"]
  F --> F2["安全策あり"]
  F --> F3["危険領域は<br/>Opus 4.8へfallback"]
  Y --> Y1["Project Glasswing"]
  Y --> Y2["一部安全策を緩和"]
  Y --> Y3["サイバー・バイオ等の<br/>高リスク用途向け"]
```

公式発表では、Fable 5とMythos 5は「同じ基盤モデル」の別構成です[^anthropic-launch]。Fable 5は広く使うために安全策を強め、Mythos 5は信頼されたパートナー向けに一部制約を緩めた位置づけです[^anthropic-mythos]。

```mermaid
flowchart TB
  F["Claude Fable 5"] --> C["context<br/>1M tokens"]
  F --> O["max output<br/>128k tokens"]
  F --> P["price<br/>$10 / MTok input<br/>$50 / MTok output"]
  F --> T["adaptive thinking<br/>always on"]
  F --> A["API ID<br/>claude-fable-5"]
```

Docs上の仕様は、1Mコンテキスト、最大128k出力、入力$10/MTok・出力$50/MTokです[^claude-models][^claude-pricing]。

```mermaid
sequenceDiagram
  participant U as User
  participant F as Fable 5
  participant C as Safety classifier
  participant O as Opus 4.8 fallback
  U->>F: 通常の依頼
  F-->>U: Fable 5が回答
  U->>C: サイバー/バイオ等の高リスク依頼
  C->>O: fallback
  O-->>U: Opus 4.8で回答
```

Fable 5向けプロンプト文書では、危険領域の分類器とfallbackが説明されています[^fable-prompting]。この安全機構はOpenMythosの主対象ではありません。

```mermaid
flowchart LR
  L["2026-06-09<br/>Launch"] --> S["2026-06-12<br/>公式に全顧客のアクセス停止"]
  S --> R["2026-06-26/27<br/>Mythos限定復旧・Fable復旧見込み<br/>という報道"]
  R --> N["ただし記事時点では<br/>公式ページはunavailable表示"]
```

2026-06-12にAnthropicは、米国政府指令によりFable 5 / Mythos 5を全顧客向けに停止すると発表しました[^anthropic-suspend]。2026-06-27には、Fable 5復旧が近いというReuters報道がありますが、公式ページ上はこの記事時点でunavailable表示です[^reuters-restore][^anthropic-fable]。

---

## 3. OpenMythosが主張している構造

```mermaid
flowchart LR
  I["tokens"] --> E["embedding"]
  E --> P["Prelude<br/>通常Transformer層"]
  P --> R["Recurrent Block<br/>同じ層をT回ループ"]
  R --> C["Coda<br/>通常Transformer層"]
  C --> L["logits"]
  R --> A["GQA / MLA"]
  R --> M["Sparse MoE"]
  R --> H["ACT halting"]
  R --> Z["LTI stable injection"]
```

OpenMythosのREADMEは、自身を「公開研究と推測だけに基づく理論的再構成」と説明し、Anthropicとの関係がないことも明記しています[^openmythos-repo]。

```mermaid
flowchart TB
  CFG["MythosConfig"] --> OM["OpenMythos"]
  OM --> P["Prelude"]
  OM --> R["RecurrentBlock"]
  OM --> C["Coda"]
  R --> B1["TransformerBlock"]
  R --> B2["LTIInjection"]
  R --> B3["ACTHalting"]
  R --> B4["LoRAAdapter"]
  B1 --> B5["Attention<br/>GQA or MLA"]
  B1 --> B6["MoE FFN"]
```

コード上も、中心は`Prelude → RecurrentBlock → Coda`です。`RecurrentBlock`は、同一のTransformerBlockを複数回使う設計として実装されています[^openmythos-main]。

---

## 4. 公式情報とOpenMythosを照合する

```mermaid
flowchart TB
  subgraph Official["公式で確認できること"]
    O1["同じ基盤モデル"]
    O2["1M context / 128k output"]
    O3["adaptive thinking"]
    O4["安全分類器とfallback"]
    O5["内部アーキテクチャは非公開"]
  end
  subgraph Open["OpenMythosの実装"]
    P1["RDT / looped depth"]
    P2["MLA or GQA"]
    P3["Sparse MoE"]
    P4["ACT / LTI"]
    P5["事前学習済み重みなし"]
  end
  O1 -.直接照合不可.-> P1
  O2 -.周辺仕様は近い.-> P1
  O3 -.挙動の説明候補.-> P4
  O4 -.未再現.-> P5
  O5 -.決定打なし.-> P1
```

ここが重要です。Fable 5の公開情報は、性能・安全策・価格・入出力仕様が中心です。内部構造は出ていません。したがって、OpenMythosのRDT仮説が「本当に同じ」とは言えません。

---

## 5. 研究として筋がある部分

```mermaid
flowchart LR
  RDT["RDT / Looped Transformer"] --> P["仮説としての土台"]
  UT["Universal Transformer / ACT"] --> P
  ST["stable looped model"] --> P
  MLA["MLA / MoE"] --> P
  P --> OK["OpenMythosの設計は<br/>研究文脈では自然"]
  OK -.ただし.-> NG["Anthropicが採用した証拠ではない"]
```

RDTは、同じTransformer層を反復して推論計算を増やす設計です。近年の研究では、反復回数を増やすことで深い推論に寄与する可能性が報告されています[^rdt-paper]。また、Universal TransformerのACT、安定なループモデル、MLA/MoEも既存研究・公開モデルに土台があります[^ut-paper][^parcae-paper][^deepseek-v2]。

---

## 6. 「正しいか」を5段階で見る

```mermaid
flowchart TB
  A["公式モデルとの一致証拠"] --> A1["1 / 5<br/>内部構造が非公開"]
  B["研究仮説としての自然さ"] --> B1["4 / 5<br/>部品は妥当"]
  C["コードの完成度"] --> C1["3 / 5<br/>Alpha相当"]
  D["実運用モデルとしての再現性"] --> D1["1 / 5<br/>重み・学習・評価がない"]
  E["教材としての価値"] --> E1["4 / 5<br/>構造理解に向く"]
```

READMEの例には`mythos_7b()`が出ますが、`__init__.py`で公開されているvariantは`1b / 3b / 10b / 50b / 100b / 500b / 1t`です[^openmythos-init]。このような小さな不整合も、公式再現ではなく研究実装として扱うべき理由です。

---

## 7. 一番わかりやすいラベル

```mermaid
flowchart LR
  A["OpenMythos"] --> B{"何として読む？"}
  B -->|正しい| C["公式Claude実装<br/>❌ NG"]
  B -->|まあ正しい| D["公開研究ベースの<br/>仮説実装<br/>✅ OK"]
  B -->|便利| E["RDT / MoE / MLAを学ぶ教材<br/>✅ OK"]
  B -->|危険| F["Fable 5互換モデルとして使う<br/>❌ NG"]
```

**結論:** OpenMythosは「Anthropicが作ったFable 5 / Mythos 5の正解」ではありません。けれど、Fable 5が見せる長時間・暗黙推論っぽい挙動を、RDTで説明しようとする仮説としては読みごたえがあります。

---

## 8. 読者が自分で確認する最短手順

```bash
git clone https://github.com/kyegomez/OpenMythos.git
cd OpenMythos

grep -R "class RecurrentBlock\|class LTIInjection\|class OpenMythos" -n open_mythos
```

```mermaid
flowchart LR
  C["clone"] --> R["READMEのdisclaimer確認"]
  R --> M["main.pyのRecurrentBlock確認"]
  M --> V["variants.py確認"]
  V --> J["公式情報と照合"]
  J --> X["公式再現ではなく<br/>仮説実装と判断"]
```

---

## 9. まとめ

```mermaid
flowchart TB
  K["覚えることは3つ"]
  K --> A["1. Fable 5 / Mythos 5は同じ基盤モデルの別構成"]
  K --> B["2. OpenMythosはRDT中心の推測実装"]
  K --> C["3. 正しい公式再現とは判断できない"]
```

**一言で:** OpenMythosは「正解」ではなく「仮説地図」。本物の地形はAnthropicがまだ公開していません。

---

## 参考資料

[^anthropic-launch]: Anthropic, “Claude Fable 5 and Claude Mythos 5” https://www.anthropic.com/news/claude-fable-5-mythos-5
[^anthropic-fable]: Anthropic, “Claude Fable” https://www.anthropic.com/claude/fable
[^anthropic-mythos]: Anthropic, “Claude Mythos” https://www.anthropic.com/claude/mythos
[^anthropic-suspend]: Anthropic, “Statement on the US government directive to suspend access to Fable 5 and Mythos 5” https://www.anthropic.com/news/fable-mythos-access
[^claude-models]: Anthropic Claude Platform Docs, “Models overview” https://platform.claude.com/docs/en/about-claude/models/overview
[^claude-pricing]: Anthropic Claude Platform Docs, “Pricing” https://platform.claude.com/docs/en/about-claude/pricing
[^fable-prompting]: Anthropic Claude Platform Docs, “Claude Fable 5 のプロンプティング” https://platform.claude.com/docs/ja/build-with-claude/prompt-engineering/prompting-claude-fable-5
[^reuters-restore]: Reuters, “US close to allowing Anthropic to restore Fable 5 model, Axios reports” https://www.reuters.com/business/us-close-allowing-anthropic-restore-fable-5-model-axios-reports-2026-06-27/
[^openmythos-repo]: kyegomez/OpenMythos, GitHub repository https://github.com/kyegomez/OpenMythos
[^openmythos-main]: kyegomez/OpenMythos, `open_mythos/main.py` https://github.com/kyegomez/OpenMythos/blob/main/open_mythos/main.py
[^openmythos-init]: kyegomez/OpenMythos, `open_mythos/__init__.py` https://github.com/kyegomez/OpenMythos/blob/main/open_mythos/__init__.py
[^rdt-paper]: Harsh Kohli et al., “Loop, Think, & Generalize: Implicit Reasoning in Recurrent-Depth Transformers” https://arxiv.org/html/2604.07822v1
[^ut-paper]: Mostafa Dehghani et al., “Universal Transformers” https://arxiv.org/abs/1807.03819
[^parcae-paper]: “Parcae: Scaling Laws For Stable Looped Language Models” https://arxiv.org/abs/2604.12946
[^deepseek-v2]: DeepSeek-AI, “DeepSeek-V2: A Strong, Economical, and Efficient Mixture-of-Experts Language Model” https://arxiv.org/abs/2405.04434
