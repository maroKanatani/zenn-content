---
title: "S3のデータを扱うアプリケーションを設計するときに考えたいこと"
emoji: "🪣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "s3", "lambda"]
published: false
---

:::message alert
下書きです
:::

扱うデータのサイズは？ 
  Lambda -> 6 MBを超えるものは 署名付きURL
  app-runnerとかの方が制限は少ないかも？

いつ・どこでzip化するのか Front? Serverside? S3イベント？
Frontend -> Serverside -> S3 Bucket




