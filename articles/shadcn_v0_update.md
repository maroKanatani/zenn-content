---
title: "shadcn add コマンドを使って v0 で作成したコンポーネントを追加してみた"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["shadcnui", "v0", "vercel", "react", "frontend"]
published: true
---

[shadcn/ui](https://github.com/shadcn-ui/ui) でパワフルなアップデートが発表されました。

https://x.com/shadcn/status/1829646543647097016

なんと [v0](https://v0.dev/) で作成したコンポーネントをプロジェクトへ直接追加できるようになったとのことです。

個人的に興奮する内容だったので、どんな感じなのか手元で試してみました。

## v0 でコンポーネントを作成する

まず、[v0.dev/chat](https://v0.dev/chat) にアクセスします。※ Vercel のアカウントが必要です。

早速コンポーネントを作成します。今回は非同期でデータフェッチする Auto Suggest というやや複雑なコンポーネントを作成してもらいました。

![v0 でのチャット画面](/images/shadcn_v0_update/01_v0_chat.png =700x)

Preview 画面でも問題なく動作していそうで、いい感じにコンポーネントを作成してくれました。

![作成されたコンポーネント](/images/shadcn_v0_update/02_v0_created.png =700x)

問題なさそうであれば、右上の Publish から Confirm and Publish を押して準備完了です。

![Confirm and Publish](/images/shadcn_v0_update/03_confirm.png =700x)

## shadcn add で追加する

Publish すると `https://v0.build/xxxxxxx` のような一意の URL が生成されます。この URL にアクセスすると `https://v0.dev/chat/b/xxxxxxx` のような URL へリダイレクトされます。

このリダイレクト後の URL を shadcn add に渡してみます。

```
$ npx shadcn add https://v0.dev/chat/b/xxxxxxx
✔ Checking registry.
✔ Installing dependencies.
✔ The file input.tsx already exists. Would you like to overwrite? … yes
✔ The file button.tsx already exists. Would you like to overwrite? … yes
✔ Created 1 file:
  - src/components/ui/suggest-component.tsx
ℹ Updated 2 files:
  - src/components/ui/input.tsx
  - src/components/ui/button.tsx
```

これで自分のプロジェクトにも追加できました。

## 微調整
実際にプロジェクトに取り込んでみてから気づいたのですが、自分のプロジェクトでは少しだけ微調整が必要でした。
- [components.json](https://ui.shadcn.com/docs/components-json) でデフォルトではない格納先を指定していたので、そのまま取り込むとエラーが発生した。
- 厳しめの `tsconfig` の設定だったので型エラーが発生した。

今回は試せていませんが、このあたりもあらかじめ `v0` の方でいい感じのプロンプトを指定できたりするとスムーズなのかもしれません。

## まとめ
`v0` とアップデートされた `shadcn add` コマンドを試してみました。
`v0` と `shadcn/ui` がシームレスに統合されることで、フロントエンドの開発効率が大幅に向上する可能性を感じました。

まだまだ発展途上な部分もありますが、これからのアップデートに大いにも期待が高まりますね。今後も `shadcn/ui` の新しい機能や改善点を追いかけていきたいです。
