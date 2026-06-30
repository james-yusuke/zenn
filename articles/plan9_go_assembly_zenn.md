---
title: "図で理解する Plan 9 アセンブリと Go の中の Plan 9"
emoji: "🪐"
type: "tech"
topics: ["go", "plan9", "assembly", "compiler", "os"]
published: false
---

Goで `.s` ファイルを書くと、`TEXT ·Add(SB), NOSPLIT, $0-24` のような見慣れない記法に出会います。これは「Plan 9というOSそのものを使っている」という意味ではなく、Plan 9系アセンブラの入力スタイルを受け継いだ、Go専用アセンブラを使うという意味です。[^go-asm]

この記事では、細かい命令表よりも「なぜそう見えるのか」を図でつかみます。

## この記事の地図

```mermaid
flowchart TD
  A["Plan 9 の歴史"] --> B["Plan 9 の設計思想"]
  B --> C["Plan 9 アセンブラ"]
  C --> D["Go のアセンブラ"]
  D --> E["TEXT / SB / FP / SP"]
  E --> F["Go から呼ぶ小さな例"]
  F --> G["今の Plan 9 と学びどころ"]
```

## Plan 9とは

Plan 9は、UnixやC言語を生んだBell Labsで1980年代後半から開発された研究用OSです。中心にある発想は「分散環境でも、資源をファイルのように扱えるようにする」ことです。プロセスごとの名前空間、9P、UTF-8、`rio`、`acme` などが特徴です。[^plan9-overview]

```mermaid
flowchart LR
  U["Unix の経験"] --> P["Plan 9"]
  P --> N["プロセスごとの名前空間"]
  P --> F["資源をファイルとして見せる"]
  P --> P9["9P: ファイルプロトコル"]
  P --> T["UTF-8 をシステム全体へ"]
  P --> W["rio / acme などの環境"]
```

## 歴史をざっくり見る

Plan 9は「Unixの後継OS」というより、Unixで得たアイデアをネットワーク時代に向けて再設計した実験場でした。

```mermaid
flowchart LR
  A["1960s-70s\nUnix"] --> B["late 1980s\nPlan 9 開発開始"]
  B --> C["1992\n第1版"]
  C --> D["1995\n第2版"]
  D --> E["2000\n第3版\nWebで配布"]
  E --> F["2002\n第4版\n9P2000"]
  F --> G["2021\nPlan 9 Foundation\nMIT License"]
```

## UnixとPlan 9の見方の違い

Unixも「everything is a file」と言われますが、Plan 9はその考えをネットワーク、ウィンドウ、プロセス、認証などへさらに押し広げます。

```mermaid
flowchart TB
  subgraph U["Unix 的な見方"]
    U1["ローカルファイル"] --> U2["プロセス"]
    U2 --> U3["ソケット"]
    U2 --> U4["端末"]
  end

  subgraph P["Plan 9 的な見方"]
    P1["/net"] --> P2["ネットワーク"]
    P3["/dev"] --> P4["デバイス"]
    P5["/mnt/wsys"] --> P6["ウィンドウ"]
    P7["9P server"] --> P8["リモート資源"]
  end
```

## 名前空間がプロセスごとに違う

Plan 9では、各プロセスが自分用の名前空間を持てます。同じ `/mnt/data` でも、プロセスAとBで別のファイルサーバを見せられます。

```mermaid
flowchart LR
  subgraph A["プロセスAの名前空間"]
    A1["/mnt/data"] --> A2["File Server A"]
    A3["/net"] --> A4["local network"]
  end

  subgraph B["プロセスBの名前空間"]
    B1["/mnt/data"] --> B2["File Server B"]
    B3["/net"] --> B4["remote network"]
  end
```

## 9Pの役割

9Pは「ファイルサーバと話すための小さな共通語」です。ローカルでもリモートでも、同じように `open/read/write` の感覚で扱えるのがPlan 9らしさです。

```mermaid
flowchart TD
  C["Client process"] -->|"walk / open / read / write"| P9["9P"]
  P9 --> FS["File server"]
  P9 --> NET["Network service"]
  P9 --> WIN["Window system"]
  P9 --> TIME["Dump / history"]

  FS --> V1["ファイルに見える"]
  NET --> V2["ファイルに見える"]
  WIN --> V3["ファイルに見える"]
  TIME --> V4["ファイルに見える"]
```

## Plan 9は今どうなったのか

Plan 9は主流デスクトップOSにはなりませんでした。一方で、公式系のコードはPlan 9 Foundationへ移り、2021年以降はMIT Licenseで公開されています。さらに、9frontなどのコミュニティフォークも継続しています。[^plan9-overview] [^ninefront]

```mermaid
flowchart TD
  A["Plan 9 from Bell Labs"] --> B["Plan 9 Foundation"]
  B --> C["MIT License"]
  A --> D["9legacy"]
  A --> E["9front"]
  A --> F["研究・趣味・教育"]
  A --> G["思想の継承"]

  G --> H["9P"]
  G --> I["UTF-8"]
  G --> J["namespace"]
  G --> K["Go assembler の見た目"]
```

## Plan 9アセンブリとは

Plan 9アセンブラは、各CPUごとに別々のアセンブラを持ちながら、共通した書き方を持つように設計されました。Rob Pikeのマニュアルでは、複数アーキテクチャ向けアセンブラが「1つのプログラムのバリエーション」のように説明されています。[^p9-asm]

```mermaid
flowchart LR
  A["Plan 9 assembler family"] --> B["MIPS"]
  A --> C["SPARC"]
  A --> D["386 / AMD64"]
  A --> E["PowerPC"]
  A --> F["ARM"]

  A --> G["共通の雰囲気"]
  G --> H["左から右へ代入"]
  G --> I["疑似レジスタ"]
  G --> J["MOVE などの擬似命令"]
```

## GoのアセンブラはPlan 9そのものではない

GoのアセンブラはPlan 9アセンブラの入力スタイルをベースにしています。ただし、Goのツールチェーン用に作られた別プログラムであり、実CPU命令をそのまま書くというより、やや抽象化された命令列を扱います。[^go-asm]

```mermaid
flowchart TD
  A["Go source"] --> B["compiler"]
  B --> C["semi-abstract instructions"]
  S[".s file"] --> AS["Go assembler"]
  C --> L["linker"]
  AS --> L
  L --> M["machine code"]

  P["Plan 9 style"] --> AS
```

## なぜ見た目が独特なのか

Goアセンブリの独特さは、主にこの4つから来ます。

```mermaid
flowchart TB
  A["Go assembly の見た目"] --> B["TEXT / DATA / GLOBL"]
  A --> C["SB / FP / SP / PC"]
  A --> D["左から右へのデータ移動"]
  A --> E["Go runtime との約束"]

  B --> B1["シンボルを定義する"]
  C --> C1["本物ではない疑似レジスタ"]
  D --> D1["MOVQ src, dst"]
  E --> E1["GC / stack / ABI"]
```

## `TEXT`行の読み方

最初に読むべき行は `TEXT` です。ここに「関数名」「スタック分割の扱い」「ローカルフレームサイズ」「引数と戻り値のサイズ」がまとまっています。

```mermaid
flowchart TD
  L["TEXT ·Add(SB), NOSPLIT, $0-24"]
  L --> A["·Add(SB)\n現在のパッケージの Add"]
  L --> B["NOSPLIT\nstack split check を入れない"]
  L --> C["$0\nlocal frame size"]
  L --> D["-24\nargs + return area"]
```

## 疑似レジスタを図で覚える

`SB`、`FP`、`SP`、`PC` は、CPUの物理レジスタというより、Goツールチェーンが管理する仮想的な目印です。

```mermaid
flowchart TB
  A["Pseudo registers"] --> SB["SB: static base"]
  A --> FP["FP: frame pointer"]
  A --> SP["SP: stack pointer"]
  A --> PC["PC: program counter"]

  SB --> SB1["·Add(SB)"]
  SB --> SB2["globalData(SB)"]
  FP --> FP1["x+0(FP)"]
  FP --> FP2["ret+16(FP)"]
  SP --> SP1["tmp-8(SP)"]
  PC --> PC1["JMP loop"]
```

## `FP`まわりの見え方

たとえば `func Add(x, y uint64) uint64` をスタック引数として見ると、`x`、`y`、`ret` が `FP` からのオフセットとして並びます。

```mermaid
flowchart TB
  subgraph Frame["caller frame から見た args/results"]
    X["x+0(FP)\nuint64"] --> Y["y+8(FP)\nuint64"]
    Y --> R["ret+16(FP)\nuint64"]
  end

  T["TEXT ·Add(SB), NOSPLIT, $0-24"] --> X
  T --> Y
  T --> R
```

## 左から右へ読む

Plan 9系の書き方では、データ移動は「左がソース、右が宛先」と読むのが基本です。Intel記法に慣れていると最初に混乱しやすいところです。

```mermaid
flowchart LR
  A["MOVQ x+0(FP), AX"] --> B["x+0(FP) を読む"]
  B --> C["AX に入る"]

  D["ADDQ y+8(FP), AX"] --> E["AX = AX + y"]
```

## Goから呼べる最小例

Go側には関数宣言だけを書き、実装を `.s` に置きます。Goのプロトタイプは、`go vet` のチェックやGCのための情報にも関係します。[^go-asm]

```go:add.go
package add

func Add(x, y uint64) uint64
```

```asm:add_amd64.s
#include "textflag.h"

TEXT ·Add(SB), NOSPLIT, $0-24
    MOVQ x+0(FP), AX
    ADDQ y+8(FP), AX
    MOVQ AX, ret+16(FP)
    RET
```

## この例の流れ

```mermaid
flowchart TD
  G["Go: Add(2, 3)"] --> P["prototype: func Add(x, y uint64) uint64"]
  P --> A["asm: TEXT ·Add(SB)"]
  A --> B["MOVQ x+0(FP), AX"]
  B --> C["ADDQ y+8(FP), AX"]
  C --> D["MOVQ AX, ret+16(FP)"]
  D --> R["return 5"]
```

## `.go` と `.s` はビルド時に合流する

パッケージに `.s` ファイルがある場合、`go build` は `go_asm.h` を生成し、Goの定数・構造体サイズ・フィールドオフセットなどをアセンブリ側から参照できるようにします。[^go-asm]

```mermaid
flowchart TD
  GO[".go files"] --> C["go compiler"]
  C --> H["go_asm.h"]
  S[".s files"] --> A["go tool asm"]
  H --> A
  C --> O1["Go object"]
  A --> O2["Asm object"]
  O1 --> L["linker"]
  O2 --> L
  L --> BIN["binary"]
```

## `TEXT`、`DATA`、`GLOBL`の位置づけ

関数は `TEXT`、初期化付きデータは `DATA`、グローバルシンボル宣言は `GLOBL` で表します。

```mermaid
flowchart TB
  A["assembly source"] --> T["TEXT"]
  A --> D["DATA"]
  A --> G["GLOBL"]

  T --> T1["function body"]
  D --> D1["bytes at offset"]
  G --> G1["symbol size / flags"]

  D1 --> G1
```

## GCとの約束

GoではGCがポインタ位置を知る必要があります。アセンブリはGoコンパイラの型安全な世界を一部抜けるので、「プロトタイプを書く」「ポインタを含まないデータには `NOPTR` を付ける」などの約束が重要です。[^go-asm]

```mermaid
flowchart TD
  A["Go prototype"] --> B["args/results の pointer 情報"]
  B --> C["GC が stack を正しく見る"]

  D["DATA / GLOBL"] --> E{"pointer を含む?"}
  E -->|"含まない"| F["NOPTR / RODATA"]
  E -->|"含む"| G["Go側で定義するのが安全"]

  F --> H["GC scan を省ける"]
  G --> I["型情報をGoに任せる"]
```

## どんな時に書くべきか

手書きアセンブリは強力ですが、読みやすさ・移植性・安全性を失いやすいです。基本はGoで書き、必要が明確な部分だけに閉じ込めるのが現実的です。

```mermaid
flowchart TB
  Q{"アセンブリを書く?"}
  Q -->|"普通の処理"| GO["Goで書く"]
  Q -->|"特殊命令が必要"| ASM["asmを検討"]
  Q -->|"SIMD / crypto / runtime"| ASM
  Q -->|"性能差を測っていない"| GO

  ASM --> R1["小さく閉じる"]
  ASM --> R2["Go prototype を置く"]
  ASM --> R3["ベンチマークを書く"]
  GO --> R4["保守しやすい"]
```

## Plan 9からGoへ、何が残ったのか

Plan 9の思想そのものがGoに丸ごと入っているわけではありません。しかし、低レイヤを見ると、Plan 9の「資源を単純なモデルで扱う」「ツールチェーンで抽象化する」という空気が残っています。

```mermaid
flowchart LR
  P["Plan 9"] --> A["assembler style"]
  P --> N["namespace thinking"]
  P --> U["UTF-8 culture"]
  P --> T["small sharp tools"]

  A --> G["Go asm syntax"]
  N --> I["system design ideas"]
  U --> S["string/text assumptions"]
  T --> C["go toolchain"]
```

## まとめ

Plan 9アセンブリを理解するコツは、CPU命令から入るより先に、`TEXT`、`SB`、`FP`、`SP` という「Goツールチェーンとの約束」を読むことです。

```mermaid
flowchart TD
  A["Plan 9"] --> B["分散OSの実験"]
  A --> C["Plan 9 assembler"]
  C --> D["Go assembler の入力スタイル"]
  D --> E["TEXT / DATA / GLOBL"]
  D --> F["SB / FP / SP / PC"]
  D --> G["GC / stack との協調"]

  H["読む順番"] --> E
  E --> F
  F --> G
  G --> I["必要な部分だけ書く"]
```

「GoでPlan 9を使っている」という言い方は少し雑です。より正確には、GoのアセンブラにはPlan 9アセンブラ由来の表記と設計思想が残っている、という理解が近いです。

## 参考資料

[^plan9-overview]: Plan 9 Foundation / 9p.io, “Plan 9 from Bell Labs — Overview”. Plan 9の概要、リリース史、2021年以降のMIT License化、現在の状態について説明しています。https://9p.io/plan9/about.html
[^p9-asm]: Rob Pike, “A Manual for the Plan 9 assembler”. Plan 9アセンブラの疑似レジスタ、`SB`、左から右へのオペランド順などを説明しています。https://9p.io/sys/doc/asm.html
[^go-asm]: The Go Programming Language, “A Quick Guide to Go's Assembler”. GoアセンブラがPlan 9アセンブラの入力スタイルに基づくこと、疑似レジスタ、`TEXT`、`DATA`、`GLOBL`、GC連携などを説明しています。https://go.dev/doc/asm
[^ninefront]: OSNews, “9front Release released”, 2025-10-13. 9frontがPlan 9のメンテナンスされているフォークであり、2025年にもリリースがあったことを紹介しています。https://www.osnews.com/story/143526/9front-release-released/
