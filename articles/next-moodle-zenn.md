---
title: "Moodleが使いにくい！"
emoji: "🎓"
type: "tech"
topics: ["nextjs","react","typescript","moodle","oss"]
published: false
---

# Moodle、正直使いにくくないですか？

大学でMoodleを使っている人なら、一度は

> 「どこ押せばいいの…？」

と思ったことがあるはずです。

課題、テスト、お知らせ、フォーラム…
全部バラバラの画面にあって、慣れるまで結構大変です。

そこで、

**「バックエンドはMoodleのまま、フロントエンドだけ全部作り直せばいいのでは？」**

という発想で作り始めたのが **next-moodle** です。

---

# コンセプト

- Moodleはそのまま使う
- UIだけNext.jsでモダン化
- 学生が迷わない画面にする
- OSSとしてみんなで育てる

---

# 技術構成

- Next.js 16
- React 19
- TypeScript
- Tailwind CSS
- Bun
- Moodle Web Service API

```text
ブラウザ
    │
Next.js(BFF)
    │
 Moodle API
```

サーバー側でAPIをまとめるので、ブラウザへMoodleのトークンを直接渡さなくても済む設計です。

---

# 作ろうとしているUI

例えばホーム画面は

- 今日やること
- 締切が近い課題
- 新着のお知らせ
- 最近の授業

が最初に見えるようにしたいです。

「まず何をすればいいか」が一瞬で分かる画面を目指しています。

---

# AIも少しだけ

AIでレポートを書く、ではなく

- 文章チェック
- 誤字脱字
- 文章を読みやすくする

くらいのサポートを想定しています。

---

# OSSなのでPR大歓迎！

まだまだ作り始めたばかりです。

UIデザインでも、
バグ修正でも、
機能追加でも、

どんなPRでも歓迎です！

リポジトリはこちら👇

https://github.com/james-yusuke/next-moodle

一緒に**Moodleをもっと使いやすく**していけたら嬉しいです 🚀
