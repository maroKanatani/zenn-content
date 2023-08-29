---
title: "【AWS】プロダクション運用を見据えた VRT 実行環境の構築"
emoji: "📸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["test", "vrt", "aws", "terraform", "frontend"]
published: false
---

# はじめに
[フロントエンド開発のためのテスト入門](https://www.amazon.co.jp/dp/4798178187)を参考に Visual Regression Test を導入しました。  

段階的に VRT を学ぶことができ大変参考にさせていただいた一方で、本文中にもありますが AWS インフラの設定等は基礎的な構成となっているので、実務で導入するにあたっていくつか設定を入れました。  

本記事では具体的にどういった設定をしたのかを列挙していきます。  

参考までに `Terraform` のコードを貼っておきますが、バージョンアップ等で古くなる可能性もあるので参考程度でお願いします。

## バージョン情報
- Terraform v1.5.3
- Terraform AWS Provider 5.8.0

:::details 最終的な構成
VRT でのみ使用するインフラは `modules/vrt` のような形式でモジュール化しておき、環境毎に分離した main で呼ぶような構成にしています。  
基本的にはテスト用の環境なので、開発環境でのみ呼ぶようにします。

- 環境毎のディレクトリ

```tf:environment/dev/main.tf
# プロバイダの定義等は省略

module "vrt" {
  source = "../../modules/vrt"

  basicauth_username = local.basicauth_username
  basicauth_password = local.basicauth_password
}


module "main" {
  # アプリ自体のインフラ等
}
```

```tf:environment/dev/locals.tf
locals {
  # VRT
  basicauth_username = "user"
  basicauth_password = "password"
}
```

- `modules/vrt`

```tf:modules/vrt/iam.tf
# ------------------------------------------------------------------------------
# VRT Role
# ------------------------------------------------------------------------------
resource "aws_iam_role" "vrt_github_actions" {
  name               = "vrt-github-actions"
  assume_role_policy = data.aws_iam_policy_document.assume_role_policy.json
}

data "aws_iam_policy_document" "assume_role_policy" {
  version = "2012-10-17"
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      identifiers = ["arn:aws:iam::{ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"]
      type        = "Federated"
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:your-repository/:*"]
    }
  }
}


# ------------------------------------------------------------------------------
#  VRT Policy
# ------------------------------------------------------------------------------
resource "aws_iam_role_policy" "vrt_github_actions" {
  name   = "vrt"
  role   = aws_iam_role.vrt_github_actions.id
  policy = data.aws_iam_policy_document.vrt_github_actions.json
}

data "aws_iam_policy_document" "vrt_github_actions" {
  # see https://github.com/reg-viz/reg-suit/blob/master/packages/reg-publish-s3-plugin/README.md
  statement {
    sid    = "regsuit"
    effect = "Allow"
    actions = [
      "s3:DeleteObject",
      "s3:GetObject",
      "s3:GetObjectAcl",
      "s3:PutObject",
      "s3:PutObjectAcl",
      "s3:ListBucket"
    ]
    resources = [
      aws_s3_bucket.vrt.arn,
      "${aws_s3_bucket.vrt.arn}/*",
    ]
  }
}

```

```tf:modules/vrt/s3.tf
resource "aws_s3_bucket" "vrt" {
  bucket_prefix = "vrt"
}

resource "aws_s3_bucket_policy" "vrt" {
  bucket = aws_s3_bucket.vrt.id
  policy = data.aws_iam_policy_document.vrt.json
}

data "aws_iam_policy_document" "vrt" {
  statement {
    sid    = "Allow Cloudfront"
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }
    actions = [
      "s3:GetObject"
    ]
    resources = [
      "${aws_s3_bucket.vrt.arn}/*"
    ]
    condition {
      test     = "StringEquals"
      variable = "aws:SourceArn"
      values   = [aws_cloudfront_distribution.vrt.arn]
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "vrt" {
  bucket = aws_s3_bucket.vrt.id

  rule {
    # 約半年
    id     = "181日経過で削除"
    status = "Enabled"
    expiration {
      days = 180
    }
    noncurrent_version_expiration {
      noncurrent_days = 1
    }
  }
}

```

```tf:modules/vrt/cloudfront.tf
resource "aws_cloudfront_distribution" "vrt" {
  origin {
    domain_name = aws_s3_bucket.vrt.bucket_regional_domain_name
    origin_id   = aws_s3_bucket.vrt.id

    origin_access_control_id = aws_cloudfront_origin_access_control.vrt.id
  }

  enabled             = true
  default_root_object = "index.html"
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD", "OPTIONS"]
    target_origin_id = aws_s3_bucket.vrt.id

    forwarded_values {
      query_string = true
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 60
    default_ttl            = 60
    max_ttl                = 60
    compress               = true

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.basic_auth.arn
    }
  }
  restrictions {
    geo_restriction {
      restriction_type = "none"
      locations        = []
    }
  }
  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

resource "aws_cloudfront_origin_access_control" "vrt" {
  name                              = "vrt"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# Basic 認証を行う CloudFront Function
resource "aws_cloudfront_function" "basic_auth" {
  name    = "vrt-website-basicauth"
  runtime = "cloudfront-js-1.0"
  publish = true
  code = templatefile(
    "${path.module}/function/basic-auth.js",
    {
      authString = base64encode("${var.basicauth_username}:${var.basicauth_password}")
    }
  )
  lifecycle {
    create_before_destroy = true
  }
}

```

```js:modules/vrt/function/basic-auth.js
function handler(event) {
  var request = event.request;
  var headers = request.headers;

  var authString = "Basic ${authString}";

  if (
    typeof headers.authorization === "undefined" ||
    headers.authorization.value !== authString
  ) {
    return {
      statusCode: 401,
      statusDescription: "Unauthorized",
      headers: {
        "www-authenticate": { value: "Basic" },
      },
    };
  }

  return request;
}

```

- GitHub Actions workflow

```yml:vrt.yml
name: Run VRT

on: push

env:
  REG_NOTIFY_CLIENT_ID: ${{ secrets.REG_NOTIFY_CLIENT_ID }}
  AWS_BUCKET_NAME: ${{ secrets.AWS_BUCKET_NAME }}
  VRT_SITE_DOMAIN: ${{ secrets.VRT_SITE_DOMAIN }}

jobs:
  build:
    # These permissions are needed to interact with GitHub's OIDC Token endpoint.
    permissions:
      id-token: write
      contents: read
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0 # この指定がないと比較に失敗する
      - uses: actions/setup-node@v3
        with:
          node-version: 18
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          role-session-name: ${{ github.event.repository.name }}-${{ github.run_id }}
          aws-region: ap-northeast-1
      - name: Install dependencies
        run: npm ci
      - name: Buid Storybook
        run: npm run storybook:build
      - name: Run Storycap
        run: npm run vrt:snapshot
      - name: Run reg-suit
        run: npm run vrt:run
```
:::

# GitHub Actions OIDC を使用して AWS のリソースへアクセスするようにする
アクセスキーを利用すると常に漏洩のリスクを気にする必要が出てきますので、管理するアクセスキーは少ないに越したことはありません。  
`reg-suit` を使うと `PutObject` や `DeleteObject` 等の更新系の操作も許容することになるので、漏洩時のリスクも小さくはないかと思っています。
検証時はともかく、本格的に運用するのであれば、なるべく早い段階で導入しておくべきでしょう。

https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect

## 1. ID プロバイダと IAM ロールの作成
まずは以下の記事をベースに ID プロバイダと IAM ロールを作成します。
https://zenn.dev/kou_pg_0131/articles/gh-actions-oidc-aws

IAM ロールには [reg-publish-s3-plugin で必要なポリシー](https://github.com/reg-viz/reg-suit/blob/master/packages/reg-publish-s3-plugin/README.md#iam-role-policy)を設定する必要があります。

  ```
      "Action": [
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:GetObjectAcl",
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:ListBucket"
      ]
  ```

:::details Terraform で記述するとこんな感じ（IAM ロール部分）
```tf iam.tf
r# ------------------------------------------------------------------------------
# VRT Role
# ------------------------------------------------------------------------------
resource "aws_iam_role" "vrt_github_actions" {
  name               = "vrt-github-actions"
  assume_role_policy = data.aws_iam_policy_document.assume_role_policy.json
}

data "aws_iam_policy_document" "assume_role_policy" {
  version = "2012-10-17"
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      identifiers = ["arn:aws:iam::{ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"]
      type        = "Federated"
    }
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:your-repository/:*"]
    }
  }
}


# ------------------------------------------------------------------------------
#  VRT Policy
# ------------------------------------------------------------------------------
resource "aws_iam_role_policy" "vrt_github_actions" {
  name   = "vrt"
  role   = aws_iam_role.vrt_github_actions.id
  policy = data.aws_iam_policy_document.vrt_github_actions.json
}

data "aws_iam_policy_document" "vrt_github_actions" {
  # see https://github.com/reg-viz/reg-suit/blob/master/packages/reg-publish-s3-plugin/README.md
  statement {
    sid    = "regsuit"
    effect = "Allow"
    actions = [
      "s3:DeleteObject",
      "s3:GetObject",
      "s3:GetObjectAcl",
      "s3:PutObject",
      "s3:PutObjectAcl",
      "s3:ListBucket"
    ]
    resources = [
      aws_s3_bucket.vrt.arn,
      "${aws_s3_bucket.vrt.arn}/*",
    ]
  }
}
```
:::


## 2. GitHub Secrets への登録
IAM ロールが作成できたら GitHub Secrets に 1 で作成した IAM Role の ARN を設定します。
（直参照が許容できるのであればそれでも可です。）

![](/images/vrt_on_aws_infra/01_vrt-github-secrets.png =700x)

## 3. workflow の修正
workflow を以下のように修正します。（一部抜粋）

```diff yml:vrt.yml
jobs:
  build:
+  # GitHub OIDC を使用するために必要な権限設定
+  permissions:
+    id-token: write
+    contents: read
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0 # この指定がないと比較に失敗する
    - uses: actions/setup-node@v3
      with:
        node-version: 18
-   - uses: aws-actions/configure-aws-credentials@master
+   # バージョンも固定しておく
+   - uses: aws-actions/configure-aws-credentials@v2
      with:
-       aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
-       aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
+       role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
+       role-session-name: ${{ github.event.repository.name }}-${{ github.run_id }}
+       aws-region: ap-northeast-1
```

# S3へはCloudfront経由でのみアクセスするようにする
S3 のアクセス周りは緩めに設定してしまうと何かと事故の元なので、Cloudfront 経由のみでアクセスできるようにしておきます。

https://blog.flatt.tech/entry/s3_security

## 1. Cloudfront の設定

Cloudfront ディストリビューションと Cloudfront Origin Access Control の作成します。

筆者は `Terraform` で作成しましたが、マネジメントコンソールで確認すると以下のようになっていました。

![](/images/vrt_on_aws_infra/cloudfront-setting.png =700x)
*Cloudfront ディストリビューションの設定例*

![](/images/vrt_on_aws_infra/oac.png =700x)
*オリジンアクセスの設定例*

:::details Terraform で記述するとこんな感じ（Cloudfront 部分）
```tf:cloudfront.tf
resource "aws_cloudfront_distribution" "vrt" {
  origin {
    domain_name = aws_s3_bucket.vrt.bucket_regional_domain_name
    origin_id   = aws_s3_bucket.vrt.id

    origin_access_control_id = aws_cloudfront_origin_access_control.vrt.id
  }

  enabled             = true
  default_root_object = "index.html"
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD", "OPTIONS"]
    target_origin_id = aws_s3_bucket.vrt.id

    forwarded_values {
      query_string = true
      cookies {
        forward = "none"
      }
    }

    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 60
    default_ttl            = 60
    max_ttl                = 60
    compress               = true

    function_association {
      event_type   = "viewer-request"
      function_arn = aws_cloudfront_function.basic_auth.arn
    }
  }
  restrictions {
    geo_restriction {
      restriction_type = "none"
      locations        = []
    }
  }
  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

resource "aws_cloudfront_origin_access_control" "vrt" {
  name                              = "vrt"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

```
:::

※ かつては `OAI` を利用して Cloudfront -> S3 のアクセスをコントロールするのがベストプラクティスと言われていましたが、最近は `OAC` というものが出てきており、今後はこちらを使うのが良いみたいです。

https://zenn.dev/kou_pg_0131/articles/tf-cloudfront-oac


## 2. バケットポリシーの設定 & パブリックアクセスブロック
`OAC` でアクセスできるように S3 バケットにバケットポリシーを設定します。
![](/images/vrt_on_aws_infra/bucket-policy.png =700x)

ここまででパブリックアクセスが不要になりますのでブロックしておきます。

![](/images/vrt_on_aws_infra/block-public-access-01.png =700x)

![](/images/vrt_on_aws_infra/block-public-access-02.png =700x)
*パブリックアクセスブロックの設定*


:::details Terraform で記述するとこんな感じ（S3 部分抜粋）
```tf:s3.tf
resource "aws_s3_bucket" "vrt" {
  bucket_prefix = "vrt"
}

resource "aws_s3_bucket_policy" "vrt" {
  bucket = aws_s3_bucket.vrt.id
  policy = data.aws_iam_policy_document.vrt.json
}

data "aws_iam_policy_document" "vrt" {
  statement {
    sid    = "Allow Cloudfront"
    effect = "Allow"
    principals {
      type        = "Service"
      identifiers = ["cloudfront.amazonaws.com"]
    }
    actions = [
      "s3:GetObject"
    ]
    resources = [
      "${aws_s3_bucket.vrt.arn}/*"
    ]
    condition {
      test     = "StringEquals"
      variable = "aws:SourceArn"
      values   = [aws_cloudfront_distribution.vrt.arn]
    }
  }
}

```
:::

Terraform で設定するとパブリックアクセスブロック等ある程度デフォルトでよしなに設定してくれます。


:::details 補足
Terraform で OAI を削除しようとすると以下のようなエラーが発生することがあります。  

```
╷
│ Error: deleting Amazon CloudFront Origin Access Identity (XXXXXXXXXXXXXX): CloudFrontOriginAccessIdentityInUse: The CloudFront origin access identity is still being used.
│       status code: 409, request id: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxx
```

関連付けがある状態の OAI を削除できないので、一旦別の OAI もしくは OAC へ関連付けを変更した状態で `terraform apply` を実行、その後 OAI が関連付いていない状態になるので、その状態で tf ファイルから既存の OAI の定義を削除し `terraform apply` を実行、というフローをたどる必要があります。
:::

# Basic認証を入れる
Cloudfront の URL がわかれば誰でもアクセスできる状態なので、念のために Basic 認証を入れておきます。  
以下のように Cloudfront Function を使って実装します。  

![](/images/vrt_on_aws_infra/cloudfront-function-01.png)
*Cloudfront Function コードの設定*

![](/images/vrt_on_aws_infra/cloudfront-function-02.png)
*Cloudfront ディストリビューションとの関連付け*


:::details Terraform で記述するとこんな感じ（Cloudfront Function 部分抜粋）
```tf:cloudfront.tf
resource "aws_cloudfront_function" "basic_auth" {
  name    = "vrt-website-basicauth"
  runtime = "cloudfront-js-1.0"
  publish = true
  code = templatefile(
    "${path.module}/function/basic-auth.js",
    {
      authString = base64encode("${var.basicauth_username}:${var.basicauth_password}")
    }
  )
  lifecycle {
    # 既存リソースがあった場合は新しいものを作成した後削除する（関連付いている状態で削除ができないため）
    create_before_destroy = true
  }
}
```
:::


https://zenn.dev/kou_pg_0131/articles/tf-cloudfront-basicauth

## 余談: Amplify の採用を見送った話
Basic 認証が手軽に導入できる Web サイトホスティングといえば Amplify を使う方法もあります。  
ただ、Amplify を使うと基本的にコンテンツを格納する S3 バケットが隠蔽されてしまい、GitHub Actions から参照できない状態になりそうだったため、今回は採用を見送りました。  
VRT を始めとしてデータレイク的にバケットにデータを都度追加していくようなケースには向いていないかと思われます。  
（今後のアップデート等で変わる可能性はあります。）

# S3のライフサイクルルールを設定する
VRT は実行の都度 S3 にデータを追加していくので、長期間運用していると保存しているデータがどんどん増えていきます。  
それに伴い保管コストも徐々に大きくなっていきます。
初期段階で必ずしも設定しておく必要はないですが、どこかのタイミングで設定しておくのがおすすめです。

ストレージクラスを変更するなど、やり方は色々考えられますが私が担当しているプロジェクトでは以下のように 181 日（約半年）経過したデータは自動で削除するように設定しました。ここはプロジェクトの状況によって適切なものを設定しましょう。

![](/images/vrt_on_aws_infra/02_s3-lifecycle.png =700x)
*S3 ライフサイクルルールの設定例*

:::details Terraform で記述するとこんな感じ（S3 ライフサイクル部分）
```tf:s3.tf
resource "aws_s3_bucket_lifecycle_configuration" "vrt" {
  bucket = aws_s3_bucket.vrt.id

  rule {
    # 約半年
    id     = "181日経過で削除"
    status = "Enabled"
    expiration {
      days = 180
    }
    noncurrent_version_expiration {
      noncurrent_days = 1
    }
  }
}
```
:::

# reg-suit の設定を修正
最後に、もろもろ構成を変更したので `regconfig.json` と workflow にも手を加えておきます。  

## 1. Cloudfront の URL を登録する
S3 の URL が設定されていましたが、 Cloudfront の URL を参照するようにしたいので、まずは GitHub Secrets に Cloudfront の URL を登録しておきます。

![](/images/vrt_on_aws_infra/cloudfront-url.png)

## 2. workflowの修正
`vrt.yml` の env に Cloudfront の URL を格納する環境変数を設定します。

```diff vrt.yml
env:
  REG_NOTIFY_CLIENT_ID: ${{ secrets.REG_NOTIFY_CLIENT_ID }}
  AWS_BUCKET_NAME: ${{ secrets.AWS_BUCKET_NAME }}
+  VRT_SITE_DOMAIN: ${{ secrets.VRT_SITE_DOMAIN }}
```


## 3. regconfig.json の修正
ACL が不要になったので `enableACL` に `false` を設定、 Cloudfront の URL を参照するようにしたので `customDomain` を設定します。

```diff json:regconfig.json
{
    "reg-publish-s3-plugin": {
      "bucketName": "$AWS_BUCKET_NAME",
+      "enableACL": false,
+      "customDomain": "$VRT_SITE_DOMAIN"
    },
}
```

この修正により `reg-suit` の通知のリンクが切り替わります。

![](/images/vrt_on_aws_infra/reg-suit.png)

# まとめ
プロダクション運用を見据えた VRT 実行環境の構築について、実践したことを列挙してきました。  
VRT について理解した後は運用面のことも考慮していきましょう。  
特に S3 に保存するデータは運用後にいつの間にか膨れ上がってコストに直結するので気を付けたいですね。

本記事が少しでも読者の皆様のお役に立てば幸いです🙇‍♂  
