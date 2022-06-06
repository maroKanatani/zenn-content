---
title: "API Gateway × Lambdaの設定による挙動の違いを見ていく"
emoji: "🔎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws", "apigateway", "lambda", "serverless", "serverlessframewor"]
published: false
---

Serverless Frameworkでバックエンドを構築した際、API Gatewayの設定によって色々ハマったので調査してみました。

# お困りごと

API Gatewayと連携しているLambdaでHTTPヘッダーが上手く取得できないことがある。

```python:handler.py
def hello(event, context):
    # 取得できたりできなかったりする
    access_token = event["headers"]["Authorization"]

    response = {
        "statusCode": 200,
        "access_token": access_token
    }

    return response
```


# API GatewayにはREST API(v1)とHTTP API(v2)がある

HTTP APIの方が機能が少なく料金が安い。
HTTP APIで済ませられるのであればその方がお財布には優しい。
が、情報はRESTの方が多い＆多機能なので運用時の変更等には強いかも

https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/http-api-vs-rest.html
https://dev.classmethod.jp/articles/amazon-api-gateway-http-or-rest/

Serverless Frameworkを使用する場合、REST APIを使用する場合は`http`、HTTP APIを使用する場合は`httpApi`を設定します。

```yml:serverless.yml
frameworkVersion: '3'

provider:
  name: aws
  runtime: python3.8
  region: ap-northeast-1

functions:
  hello:
    handler: handler.hello
    events:
      # HTTP API(v2)を使用する場合の設定 
      - httpApi:
          path: /hello-http-api
          method: get
      # REST API(v1)を使用する場合の設定
      - http:
          path: /hello-http
          method: get
```

# eventオブジェクト内の構造を見てみる

```python:handler.py
import json


def hello(event, context):
    response = {
        "statusCode": 200,
        "event": json.dumps(event)
    }

    return response
```