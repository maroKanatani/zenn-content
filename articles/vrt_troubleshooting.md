---
title: "私がVisual Regression Testを導入する際にハマったこと & 回避策"
emoji: "🧪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["test", "storybook", "vrt", "frontend", "react"]
published: true
---

# はじめに
Visual Regression Test は画像のピクセル単位で UI の変更を監視できる強力なテスト手法です。
しかし、実際にプロジェクトに導入するにあたっていくつかの課題に直面し、その解決策を模索する必要がありました。  
この記事では、私が実際に遭遇した問題と、それらに対する回避策を共有します。  

# プロジェクトの特徴
- `Storybook` が導入されており、ほぼすべてのコンポーネントで Story が設定されている。
- テスト時は `msw` で API をモックしており、`Storybook` でも利用している。
- Storybook では `Play Function` を多用している。
  - 例えばフォームでバリデーションエラーが発生している状態を再現するのに `Play Function` で異常値を入力 -> Submit する、みたいなことをしている。
  - 純粋に Props のみに依存するようなコンポーネントは少ない。（API のレスポンス、Global State 等）
    - ここが綺麗にできているのであればハマりどころはかなり少なくなるのではないかなぁと想像しています。
- UI コンポーネントライブラリを採用している。
  - 自分たちで Atomic Design のような厳密なコンポーネント分割をするようなことはしておらず、`Organisms` 程度の粒度のコンポーネントが多い。

# 課題
VRT 導入前はスナップショットテストを行っていた（現在も稼働中）のですが、UI コンポーネントライブラリを採用していると、（見た目の変更がないのに）ライブラリ内部で管理している id や class、DOM の構造（span タグで実装されていたものが div タグになるなど）等がバージョンアップにより書き換わることがあり、それらの差分を追うのが大変、かつやはり画像のような見た目にもわかりやすい方法で比較ができるのであればその方が良いと考えるようになりました。  

# 採用した手法
最終的には `storycap` + `reg-suit` を採用することにしました。  
ただ、まだまだ情報が少ないこともあり、検証時にハマりどころの解消方法がわからないことも多かったので、`Playwright` 等を使って自前でスクリーンショットのタイミング等をコントロールすることも検討しました。

# 実際に直面したハマりどころ

## mswで `delay("infinite")` しているとstorycapでのスクリーンショット取得時にtimeoutしてしまう
Loading 状態の Story を再現するために msw で `delay("infinite")` を使ってレスポンスを待機させていたのですが、この状態で storycap を使うと timeout してしまい、スクリーンショットが取得できなかったです。
最終的にはこういったパターンの場合は `skip` 等で除外するようにしたり、以下の記事にあるような zx のスクリプトを実装し、該当するパターンを除外するなどしてから `storycap` に渡すようにしました。

https://blog.wadackel.me/2022/vrt-performance-optimize/

```ts
export const Loading: Story = {
  parameters: {
    msw: {
      handlers: [
        rest.all(`*`, (_req, res, ctx) => {
          return res(ctx.delay("infinite"));
        }),
      ],
    },
    screenshot: {
      skip: true, // 追加
    },
  },
};
```

## Storybookでchunk load errorが発生する
Storybook 起動後、`storycap` でスクリーンショットを取得しようとする際、`chunk load error` が発生することがありました。
最終的には以下の記事を参考にチャンク分けを無効にする処理を入れたことで安定するようになりました。

https://support.invisionapp.com/docs/troubleshooting-chunkloaderror-in-dsm-storybook

## rechartsのグラフ描画の終了を待たずにスクリーンショットが取得される
グラフライブラリに recharts を使っているのですが、グラフ描画時のアニメーションが終わる前にスクリーンショットが取得されてしまうことがありました。
storycap のドキュメントを見ていると `--disableCssAnimation` あり、最初はこれを設定すればよいかと考えたのですが、どうも SVG アニメーションはこの設定では無効にできないようでした。
最終的には recharts コンポーネントの `isAnimationActive` をテスト時のみ `false` に設定することで回避できました。

```tsx
<BarChart>  
  <Bar
    // ...
    // テスト時以外の場合のみ有効
    isAnimationActive={process.env.NODE_ENV !== "test"}
  >
    ...
  </Bar>
</BarChart>
```

## 一部のコンポーネントで `Play Function` の終了を待たずしてスクリーンショットが取得される。
`storycap` を採用している場合、基本的には以下の記事にあるように `Play Function` の実行を待ってからスクリーンショットが取得されます。
https://qiita.com/Quramy/items/d332e37dded505daac90

一方で、

> 以下のように、play function 内で 中途半端に delay が発生すると、現状の Storycap は「レンダリングエンジンが安定した」と誤解してキャプチャを実行してしまうかも。

とあるように Storycap がレンダリングエンジンが安定したと誤解してしまうようなパターンの場合はひと工夫必要になります。

私のプロジェクトでは明示的に `delay` を使っている部分はなかったですが、`testing-library` の `waitFor` やコンポーネントが制御コンポーネントであることから来る入力操作のパフォーマンス面でのコストなどが重なり、 `Play Function` の実行途中でスクリーンショットが取得されることがありました。

本当は根本原因を解消できるのが一番良いと思いますが、一旦は以下のように `Play Function`, `storycap` の waitFor から参照できる変数を用意してそれを確認することで回避できました。

```ts
// 何かしら変数を宣言する。
let playFunctionEnd: Promise<string>;

export const HasPlayFunction: Story = {
  play: async ({ canvasElement }) => {
    // ...

    // 最後の行で変数を更新
    playFunctionEnd = Promise.resolve("resolved");
  },
  parameters: {
    screenshot: {
      waitFor: async () => {
        // 宣言した変数を参照する
        await waitFor(async () => {
          expect(await playFunctionEnd).toBe("resolved");
        });
      },
    },
  },
};
```

# まとめ
VRT を実際に導入するにあたって私がハマったことを列挙してきました。  
VRT はかなり効果的なテスト手法である一方、閾値のチューニング等、実際に運用しながら検証していくことがかなり重要だと思います。  
おそらく実際には上記以外にも様々な課題があるのでしょうし、私としても今後運用していく上で新しい課題に直面するかもしれません。  
新しい課題があれば可能な限り随時追記していく予定です。  
本記事が少しでも読者の皆様のお役に立てば幸いです🙇‍♂  

