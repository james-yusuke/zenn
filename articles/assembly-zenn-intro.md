---
title: "図で見るアセンブリ入門：種類・書き方・x86の基本"
emoji: "🧩"
type: "tech"
topics: ["assembly", "x86", "cpu", "beginner"]
published: true
published_at: "2026-06-28 17:44"
---

## はじめに

アセンブリは、**CPUに近い言葉**です。

普通のプログラムより、CPUが何をしているかが見えます。

```plaintext
人間が書くコード
      ↓
高級言語 C / Rust / Python
      ↓
アセンブリ
      ↓
機械語 010101...
      ↓
CPUが動く
```

この記事では、深くやりすぎずに、

1. アセンブリの種類
2. アセンブリの書き方
3. x86の基本
4. label と jmp

までを紹介します。

---

## アセンブリとは

アセンブリは、**機械語を人間が読めるようにしたもの**です。

```plaintext
機械語
10111000 00000001 00000000 00000000 00000000

↓ 人間向けにすると

アセンブリ
mov eax, 1
```

イメージです。

```plaintext
CPU「数字だけだと人間が大変そう」

人間「mov とか add なら読める」

CPU「でも最後は機械語にしてね」
```

---

## アセンブリの種類

アセンブリは1種類ではありません。

**CPUの種類によって、使う命令が違います。**

| 種類 | よく使われる場所 | イメージ |
|---|---|---|
| x86 / x86-64 | Windows PC、Linux PC | パソコン系 |
| ARM / AArch64 | スマホ、Apple Silicon | 省電力で強い |
| RISC-V | 研究、教育、組み込み | オープンなCPU |
| MIPS | 教科書、古い機器 | 学習で見やすい |

図にするとこうです。

```plaintext
アセンブリ
├─ x86 / x86-64
├─ ARM / AArch64
├─ RISC-V
└─ MIPS
```

同じ「足し算」でも、CPUが違うと書き方が変わります。

```plaintext
CPUが違う
   ↓
命令の名前や形が違う
   ↓
アセンブリも違う
```

---

## 書き方にも種類がある

x86でも、書き方の流派があります。

| 記法 | よく見る場所 | 特徴 |
|---|---|---|
| Intel記法 | NASM、MASM | `mov rax, 1` のように見やすい |
| AT&T記法 | GAS、Linuxの一部 | `%rax` や `$1` が出る |

同じ意味でも、見た目が違います。

### Intel記法

```asm
mov rax, 1
```

```plaintext
rax に 1 を入れる
```

### AT&T記法

```asm
movq $1, %rax
```

```plaintext
1 を rax に入れる
```

この記事では、読みやすい **Intel記法** で書きます。

---

## アセンブリの基本形

アセンブリは、だいたいこの形です。

```plaintext
ラベル:
    命令  目的地, 材料   ; コメント
```

例です。

```asm
start:
    mov rax, 1      ; rax に 1 を入れる
    add rax, 2      ; rax に 2 を足す
```

分解するとこうです。

```plaintext
start:
│
└─ ラベル
   場所につける名前

    mov rax, 1
    │   │    │
    │   │    └─ 材料
    │   └────── 目的地
    └────────── 命令
```

---

## x86の考え方

x86では、CPUの中に小さな入れ物があります。

それが **レジスタ** です。

```plaintext
CPU
┌────────────────────┐
│ レジスタ            │
│ ┌────┐ ┌────┐       │
│ │RAX │ │RBX │ ...   │
│ └────┘ └────┘       │
└────────────────────┘
```

イメージです。

```plaintext
レジスタ = CPUのポケット
メモリ   = 大きな棚
```

CPUは、ポケットに入れた値を使って計算します。

---

## よく見るx86-64レジスタ

| レジスタ | よくある役割 |
|---|---|
| RAX | 計算結果を入れがち |
| RBX | 一時的な値を入れがち |
| RCX | 回数を数えるときに使いがち |
| RDX | 計算やデータで使いがち |
| RSP | スタックの位置 |
| RBP | 関数の基準位置 |

最初は全部覚えなくて大丈夫です。

まずはこれだけでOKです。

```plaintext
RAX = よく使う計算用の箱
```

---

## 基本命令

まずはこの4つだけ見るとわかりやすいです。

| 命令 | 意味 | 例 |
|---|---|---|
| `mov` | 入れる | `mov rax, 1` |
| `add` | 足す | `add rax, 2` |
| `sub` | 引く | `sub rax, 1` |
| `cmp` | 比べる | `cmp rax, 10` |

---

## mov：値を入れる

```asm
mov rax, 10
```

```plaintext
10 ─────▶ RAX
```

つまり、

```plaintext
RAX = 10
```

です。

---

## add：足す

```asm
mov rax, 10
add rax, 5
```

```plaintext
RAX = 10
RAX = RAX + 5
RAX = 15
```

図にするとこうです。

```plaintext
RAX
┌────┐
│ 10 │
└────┘
  │ +5
  ▼
┌────┐
│ 15 │
└────┘
```

---

## sub：引く

```asm
mov rax, 10
sub rax, 3
```

```plaintext
RAX = 10
RAX = RAX - 3
RAX = 7
```

---

## cmp：比べる

```asm
cmp rax, 10
```

これは、

```plaintext
rax と 10 を比べる
```

という意味です。

ただし、答えをRAXに入れるわけではありません。

```plaintext
cmp は「比べた結果」をCPUのメモに残す
```

ざっくり図です。

```plaintext
cmp rax, 10
      ↓
CPUの中のメモ
┌─────────────┐
│ 同じ？       │
│ 大きい？     │
│ 小さい？     │
└─────────────┘
```

この結果を使って、次の `jmp` 系の命令で移動します。

---

## label：場所に名前をつける

labelは、コードの場所につける名前です。

```asm
start:
    mov rax, 1

end:
    mov rbx, 2
```

図にするとこうです。

```plaintext
start:
  ↓
mov rax, 1
  ↓
end:
  ↓
mov rbx, 2
```

labelがあると、

```plaintext
ここに飛びたい
ここから始めたい
ここで終わりたい
```

のように、場所を指定できます。

---

## jmp：別の場所へ飛ぶ

`jmp` はジャンプです。

```asm
jmp end
```

意味は、

```plaintext
end というラベルへ飛ぶ
```

です。

例です。

```asm
start:
    mov rax, 1
    jmp end

    mov rax, 999

end:
    mov rbx, 2
```

流れです。

```plaintext
start
  ↓
mov rax, 1
  ↓
jmp end
  └──────────────┐
                 ↓
                end
                 ↓
              mov rbx, 2
```

`mov rax, 999` は飛ばされます。

```plaintext
jmp で飛ぶ
   ↓
途中の命令は実行されない
```

---

## 条件つきジャンプ

`cmp` と `jmp` 系を合わせると、if文みたいなことができます。

| 命令 | 意味 |
|---|---|
| `je` | 同じなら飛ぶ |
| `jne` | 違うなら飛ぶ |
| `jg` | 大きいなら飛ぶ |
| `jl` | 小さいなら飛ぶ |

例です。

```asm
mov rax, 10
cmp rax, 10
je same

mov rbx, 0
jmp end

same:
    mov rbx, 1

end:
```

図です。

```plaintext
RAX == 10 ?
   │
   ├─ yes → same へ飛ぶ → RBX = 1
   │
   └─ no  → RBX = 0
```

C言語っぽく書くとこうです。

```c
if (rax == 10) {
    rbx = 1;
} else {
    rbx = 0;
}
```

---

## 小さなまとめ

```plaintext
アセンブリ = CPUに近い言葉

種類
├─ x86
├─ ARM
├─ RISC-V
└─ MIPS

x86の基本
├─ mov  入れる
├─ add  足す
├─ sub  引く
├─ cmp  比べる
├─ label 場所の名前
└─ jmp  飛ぶ
```

最後に、この記事の内容を1枚にするとこうです。

```plaintext
人間のコード
   ↓
アセンブリ
   ↓
CPUの命令

CPUの中
┌──────────────┐
│ RAXなどの箱   │
└──────────────┘

命令
mov → 入れる
add → 足す
cmp → 比べる
jmp → 飛ぶ
```

アセンブリは難しく見えます。

でも最初は、

```plaintext
値を入れる
計算する
比べる
飛ぶ
```

だけ見ればOKです。
