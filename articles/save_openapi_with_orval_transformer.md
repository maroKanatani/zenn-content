---
title: "Orvalのtransformerを使ってOpenAPIの定義ファイルを書き出す"
emoji: "🐟️"
type: "tech"
topics: ["orval", "typescript", "openapi"]
published: true
---

# はじめに
[Orval](https://orval.dev/)を使うと OpenAPI からクライアントコードを生成できます。  
この際、生成に使用した OpenAPI の定義ファイルをローカルへ保持し、Git 管理したいことがありました。  
Orval では、生成されたクライアントコードのカスタマイズを可能にするために[transformer](https://orval.dev/reference/configuration/input#transformer)を使用できます。  
この機能を利用することでクライアントコードの生成時に定義ファイルを書き出すことができます。

# 前提
以下の環境を前提としています。
```sh
$ node -v         
v22.14.0
$ npm init -y
$ npm install orval --save-dev
$ npm ls orval
orval-sample@1.0.0 /path/to/orval-sample
└── orval@7.8.0
```

設定ファイルは以下のような形式です。  
input に外部 URL が設定されているため、どういった変更によってクライアントコードが変更されたのかがわからないことがありました。
そのため生成されたクライアントコードのみではなく、定義ファイル自体の差分も確認したいと考えていました。

```ts:orval.config.ts
import { defineConfig } from 'orval'

export default defineConfig({
  petstore: {
    input: 'https://petstore.swagger.io/v2/swagger.json',
    output: "generated",
  },
})
```

package.json には以下のようなスクリプトを用意しています。
```json:package.json
{
  "scripts": {
    "orval": "orval --config ./orval.config.ts"
  },
}
```

# transformerを用意する
今回は以下のような transformer を用意しました。  
本来は読み込んだ定義を変換するためのものですが、書き込みを行った後、元データをそのまま返すようにしています。
```ts:transformer.ts
import { InputTransformerFn } from "orval";
import { writeFileSync } from "fs";

export const saveOpenApiTransformer: InputTransformerFn = (spec) => {
  try {
    // OpenAPI定義をJSONとして保存
    writeFileSync("openapi.json", JSON.stringify(spec, null, 2));
    console.log("✨ OpenAPI definition saved to: openapi.json");
  } catch (error) {
    console.error("Failed to save OpenAPI definition:", error);
  }

  // 元のOpenAPI定義をそのまま返す（クライアントコード生成に使用される）
  return spec;
};
```

# configファイルの書き換え
transformer を config ファイルに設定します。

```diff ts:orval.config.ts
 import { defineConfig } from 'orval'
+import { saveOpenApiTransformer } from './transformer'

 export default defineConfig({
   petstore: {
-    input: 'https://petstore.swagger.io/v2/swagger.json',
+    input: {
+      target: 'https://petstore.swagger.io/v2/swagger.json',
+      override: {
+        transformer: saveOpenApiTransformer
+      }
+    },
     output: "generated",
   },
 })
```

これで準備が整いました。コマンドを実行して定義ファイルが出力されるか確認しましょう。
```sh
$ npm run orval
```

# まとめ
Orval の transformer を使って OpenAPI の定義をローカルに出力する方法を紹介しました。  
プロジェクトを運用していく中で、orval 自体のバージョンアップ等でもクライアントコードが変更されることがあり、OpenAPI 定義の変更の影響なのかバージョンアップによる影響なのか（もしくはその両方なのか）の判断に苦労することがありました。
今回のような対応を入れることで継続運用がやりやすくなる部分もありそうです。
