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


```
$ cdk --version
2.20.0 (build 738ef49)
```


考える仕様
- 単一のS3オブジェクトをダウンロードする
- 選択した複数のS3オブジェクトをまとめてダウンロード
- 指定したフォルダ（プレフィックス）配下にあるS3オブジェクトをまとめてダウンロードする

扱うデータのサイズは？ 
  Lambda -> 6 MBを超えるものは 署名付きURL
  app-runnerとかの方が制限は少ないかも？

いつ・どこでzip化するのか Front? Serverside? S3イベント？
  Frontend -> Serverside -> S3 Bucket

バイナリで扱うのか署名付きURL経由で扱うのか
  バイナリの場合、API GatewayやLambdaで設定 & 6MB以下のデータのみ扱う
  署名付きURLの場合、
    フロントエンドでzipパターンだと単一のS3オブジェクトをダウンロードする関数を複数回呼ぶような実装にしようとするとS3バケットのアクセスコントロールを調整しないといけなさそう
    バックエンドでzipパターンだと、圧縮したzipファイルをS3にアップロードしてから署名付きURLを発行
    望ましいのは多分S3イベントで圧縮するパターン？
  

参考
App Runner on CDK
https://dev.classmethod.jp/articles/cdk-app-runner/
