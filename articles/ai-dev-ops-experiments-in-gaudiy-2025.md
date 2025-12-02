---
title: "プロダクトのAI開発におけるラストワンマイル活用"
emoji: "🧭"
type: "tech"
topics: ["LLM", "Claude Code", "GitHub Actions", "GauDev" , "アドベントカレンダー"]
published: true
published_at: 2025-12-05 00:00
---

この記事は[#GauDev Advent Calendar 2025](https://adventar.org/calendars/11616)の5日目です。

---

今年は AI 開発が加速する1年でした。Claude Code や MCP(Model Context Protocol) が公開されたのがまだ今年だなんて、信じられませんね...

Gaudiy でも各チームで AI 開発を積極的に取り入れてきました。今回の記事では、その一部であるAI開発におけるラストワンマイル活用として試していることをご紹介します。

## AI 開発におけるラストワンマイルとは 
Claude Code をはじめとするAIコーディングエージェントの強みは、CLAUDE.mdを起点にプロダクトに関するコンテキストを持った働きができるということが挙げられます。
しかし、すべての職能の人が Claude Code を使えないため、プロダクトのコンテキストをもったAI活用ができません。[^1]

コンテキストを持っていない LLM アプリケーション(Chat GPT, Gemini, Claude など) にプロダクトのことを聞いてもハルシネーションを起こすか、分からないと回答が返ってくるだけです。

このような障壁が存在するため、以下のような業務改善に取り組めませんでした

- プロダクトの仕様確認がChat GPTをはじめとするデスクトップアプリがないとできない
- 不具合発生時に原因調査をエンジニアに毎回お願いする必要があり、担当者が属人的

[^1]: 厳密には Claude Code Web や GitHub MCP を活用してアクセスできますが、そのためには GitHub アカウントの作成や MCP の設定など障壁が多くあるため、すべての人が実現できるには現実的ではない、という考えを前提に置いています。

こういった課題が現場では起きていました。

## Claude Code Action を操る Slack Bot の導入



## おわりに



---
[#GauDev Advent Calendar 2025](https://adventar.org/calendars/11616)、明日の担当はYuseiWhiteさんです。
