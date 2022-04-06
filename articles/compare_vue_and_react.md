---
title: "Reactを使うのかVueを使うのかについて個人的なモチベーションを整理したかった"
emoji: "⚖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "vue", "typescript", "フロントエンド", "frontend"]
published: true
---

:::message alert
本記事はあくまで執筆時点(2022/3/27)での一意見でありますので、今後時間や技術的な変化により参考にならない部分も出てくるかもしれません。
:::

Reactはいいぞ、Vueはいいぞと様々な情報が世の中には溢れているものの、「こういう場合には」という前提条件にあまり言及されていない情報が多いような気がしたので自分なりの視点で考えてみたいと思いました。

また、SvelteやAngular等他のフレームワークもありますが、そちらは個人的にはよくわからないので、あくまでReactとVueについてだけ言及していきます。

# 私のフロントエンド経験と気持ちの変化
- 2018年くらいにReactを勉強し始める。
  - → Reactって難しい…。
- 2019年くらいにVueを学び始める。
  - → Vueって簡単！Reactよりわかりやすくてええやん！
- 2020年くらいにNuxtの案件に参画する。
  - → Webフロントエンドへの理解が深まる。
- 2021年〜 Vueの案件に参画（プライベートで何か作る時はReact）
  - → Vue結構辛いな…。やっぱReactがいい…。

上記のような経歴なのでVueの方が実際の現場で触っている時間が長いです。
特に最近は移行の検討を始めていたりするので、その辺りには特にフォーカスするかもしれません。

記事を全体を通して**中規模程度のフロントエンドの開発・運用をVueでやっているReact好きおじさんの意見・感想であること**をご理解いただければと思います🙇‍♂️ **（重要）**

# 書き方の比較
React vs Vueの比較もしたいのですが、Hooks・Composition APIの登場前後で同ライブラリ内でも書き方に結構違いがあるのでそこも含めて並べてみます。

## React

- Hooks以前のReact、Classベースで記述する。
```js
import React from 'react'
 
class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      count: 0
    };
  }
  increment = () => {
    this.setState({ count: this.state.count + 1 })
  }
  
  render() {
    return (
      <div>
        <p>count: {this.state.count} times</p>
        <button onClick={this.increment}>
           Click
        </button>
      </div>
    );
  }
 }
```

- 関数コンポーネントとHooksで記述、最近ほぼこちら

```js
import React, { useState } from 'react';
 
export const App = () => {
  // countはnumber型
  const [count, setCount] = useState(0); 
  const increment = () => {
    // 再代入せず必ず関数を通して状態を更新する
    setCount(count + 1); 
  }
  return (
    <div>
        <p>count: {count} times</p>
        <button onClick={increment}>
          Click
        </button>
    </div>
  );
}
```

## Vue

- Vue2 Option API

```vue
<template>
  <div>
    <p>count: {{ count }} times</p>
    <button @click=increment>
       Click
    </button>
  </div>
</template>
 
<script>
export default {
  data() {
    return {
      count: 0
    }
  },
  methods: {
    increment() {
      this.count++
    }
  }
}
</script>
```

- Vue3 script setup構文で記述

```vue
<script setup>
import { ref } from 'vue'

// countはRef<number>型
const count = ref(0) 
const increment = () => {
  // ラップされた値を更新する。再代入っぽい状態更新
  count.value++ 
}
</script>
 
<template>
  <div>
    <p>count: {{ count }} times</p>
    <button @click=increment>
       Click
    </button>
  </div>
</template>
```

Vueについては上記２つの書き方の間にも様々な記法があります。
- Vue.extend
- ClassComponent
  - Vue2でTypeScriptサポートを高めるために考案された記法だが、Vue3ではネイティブでTypeScriptサポートされるようになったので現在は化石
  - リリースも2020年でストップしており、積極的なメンテはされないように見受けられる（[リポジトリ](https://github.com/vuejs/vue-class-component)）
- setup構文…etc


ReactのClassComponentやVueのOption API等を採用する場合、[フックパターン](https://zenn.dev/morinokami/books/learning-patterns-1/viewer/hooks-pattern#%E3%82%B9%E3%83%86%E3%83%BC%E3%83%88%E3%83%95%E3%83%83%E3%82%AF)的に状態を管理できないので、単一のコンポーネントが持つ情報がどんどん肥大化し、スケールが難しくなっていく印象があります。

また、React Hooksはリリースされてからある程度時間が経過しており、ネット上でもノウハウが散見されるようになってきましたが、VueのComposition APIについてはReactよりも後発であったり、色んな事情でOption APIも結構現役な気がしています（主観を含む）。
script setup構文は結構スマートで良い感じになったと思います。

可能な限りVueを採用する場合で将来的なスケールも見据えたいという場合は、Reactでサンプルコードが書かれている記事等も参考にしつつVueに落とし込むということをしておいた方が幸せになるのではないかと思っています。
（そこまで頑張るんならReactやろうよ…って言われるとそれはそうとしか言えないけど）

↓とか
https://zenn.dev/morinokami/books/learning-patterns-1

# 個人的にVueを採用しても良いと思う場面

## 開発規模の話
個人開発〜小規模程度の開発であればアリかと思います。
作り方によっては中規模程度までもいけるかもしれません。

Vueの良い部分として直感的なAPIで導入が簡単というメリットがあると思うので、あまりフロントエンド開発になれていない人がそこに全力で乗っかり初速を高めた開発を進められるのは大きなメリットではあると思います。
（もちろん最初からReactができる人にとってはこの限りではないかもしれません）
また、状態管理ライブラリやCSSについては選択肢がReactほど多くないので、その辺りの選定コストが比較的低いところも良い部分と言えるかと思います。

一方で、ある程度のチーム人数や規模になった際にTypeScriptの型によるサポートやtscによるチェックによる恩恵を受けたいというような場合や、単方向データバインディングしかできないことによるシンプルな設計のしやすさはReactに軍配が上がるように思います。

特にVueの場合は双方向データバインディングが使えてしまうので、これを濫用してしまうとデータの流れがカオスになりスケールが難しく、末端で負債を残しやすくなる気がしています。

## 運用面での話
いつでも捨てられる・長期の運用をあまり考えなくて良い場合はアリかと思います。

というのも、長期での運用を考えた場合、現時点ではVue2 -> Vue3へのBreaking Changeや、それによって引き起こされているであろう周辺ライブラリの対応の遅れによる開発スピードの低下・移行コストの高さが無視できるレベルではないように感じているからです。

特に（超メジャーOSSの）Nuxt.jsが現在Public Betaではあるものの、[プロダクションで気軽に導入できない状態](https://v3.nuxtjs.org/getting-started/introduction)であったり、[VuetifyもまだBeta版](https://next.vuetifyjs.com/en/getting-started/installation/)なのはなかなか厳しいです。（2022年3月時点）
Vue3がリリースされたのが2020年9月なので対応に1年以上かかっています。

また、移行については一応公式が移行ビルドを公開しています。
[公式移行ビルド](https://v3.ja.vuejs.org/guide/migration/migration-build.html#アップグレードのワークフロー)

>9. vuex を v4 にアップグレード します。
>10. vuex-router を v4 にアップグレードします。

プラグイン周りの仕様の変化により、Vue3でVue2までのライブラリが使えないものがあるようなので、**コアのアップグレードと関連するライブラリのアップデートを一気に実施する必要があります。**

- [Vue2の仕様](https://jp.vuejs.org/v2/guide/plugins.html#プラグインの使用)
- [Vue3の仕様](https://v3.ja.vuejs.org/guide/plugins.html#%E3%83%95%E3%82%9A%E3%83%A9%E3%82%AF%E3%82%99%E3%82%A4%E3%83%B3%E3%82%92%E4%BD%BF%E3%81%86)

フロントエンドについてはメジャーバージョンアップでBreaking Changeが発生するライブラリはよく目にしますが、コア部分のアップグレードが他に多大な影響を与える状態はなかなか厳しいように感じます。

（まぁ頑張ればできるんだろうけどね…）

上記についてはVue2 -> Vue3移行についての話なので、Vue3で新規開発する場合はアリかもしれないのですが、Vue3 -> Vue4で同じようなことが起きないとは必ずしも言えない（起きるとも言えないですが）のと、使用できるライブラリが制限されることがあるというのは念頭に入れておいた方が良いでしょう。

VuetifyやNuxtを使いたいから新規でもVue2を採用したいという案件もたまに聞きますが、移行コストを含めて考えると極力避けることをお勧めします。
もちろん冒頭の通りPoCやプロトタイプ用途で長期運用しない前提であればこの限りではありません。

## チームメンバーの話
チームメンバーで以下のような人がいれば採用を検討しても良いかもしれません。
各々の好みや経験の問題も考慮すべきでしょう。

- もともとマークアップを担当していた人・JSよりもHTML/CSSが好きな人等がフロントエンド開発へ参画する場合
- どうしても初期学習コストをかけたくない
- どうしてもReact(JSX)に抵抗を感じて仕方がない
- Vueが好きでたまらない…etc


## 一言でまとめると
**もろもろの不安定さよりもぱっと見の簡単さを重視したい。**

といった感じでしょうか。

# Vueで辛いと思うところ

## 周辺ライブラリ状況も考慮しながらの移行
上記にも挙げたのですがエコシステムの分断もあり、周辺ライブラリも含めた移行はなかなか気合が必要かと思います。

また、多種多様な構文についてはある程度後方互換性があるのですが、Breaking Changeによって`.sync`など廃止されたものもあるので注意が必要です。

https://v3.ja.vuejs.org/guide/migration/v-model.html#%E6%A6%82%E8%A6%81

ClassComponentに関しては今後積極的にメンテされなさそうなので、仮にVue2を採用する場合、ClassComponentの採用は控えた方がよさそうです。

## 純粋なJavaScript（TypeScript）の世界からの乖離

Vueは`.vue`という独自のフォーマットで記述します。
この独自のフォーマットがそれまでHTMLのマークアップに慣れた人にとっては直感的であり、これがVueのよさでもあると思う一方で、厳密にはこの構文はHTMLでもCSSでもJavaScriptでもありません。

これによりエディタの補完等のために構文を解析しようと思うと独自の拡張機能を入れる必要があります。

これまでであればVue2時代でメジャーであった[Vetur](https://github.com/vuejs/vetur)やVue3以降で推奨されている[Volar](https://github.com/johnsoncodehk/volar)といった拡張機能が存在します。

これらはとても便利でありがたいと思いますし、解析の精度も高まってきているものの、新しいバージョンや構文が出てくるたびに拡張を変えていくのは個人的には少し手間を感じてしまいます。

また、この独自のフォーマットがあるからか、末端部分でのTypeScriptのサポートに個人的には物足りなさを感じてしまいます。

~~例えばTypeScriptの構文を以下のようにtemplate部分で使用するとSyntax Errorが発生します。~~
~~template部分はTypeScriptとして解釈されないみたいです。~~
  - **2022/04/01 追記)** scriptの方にsetupが抜けておりました。Vue3から以下のような構文で補完ができそうです。

```vue
<template>
  <button @click="handleClick(message!)"></button>
</template>

<script setup lang="ts">
import { ref } from "vue"

const message = ref<string|undefined>(undefined);
const handleClick = (message?: string) => {
  console.log(message);
}
</script>
```

（`!`を使うこと自体があまりよろしくないというツッコミは一旦無しで）

また、私はVSCodeを使って開発をしており、ピーク > 呼び出し階層のプレビューを使いたいと思うことが多いのですが、このとき`.vue`ファイルを対象に検索することができないのがそれなりの規模の開発をしていると不便に感じてしまいます。
以下はuseCounterという関数がコールされている箇所を検索しようとしているところです。

![](/images/compare_vue_and_react/image01.png =700x)
*Reactの場合、.tsx, .tsファイルが対象になっている*
![](/images/compare_vue_and_react/image02.png =700x)
*Vueの場合、.vueは検索の対象にならない*

この辺りも拡張機能があったりするかもしれないですが、個人的にはそこを頑張りたいわけじゃないんだけどな〜という気持ちになってしまいます。

VueとTypeScriptとの相性が良くないことは昔から言われていますし、Vue3でかなり改善はしているものの、やはり**TypeScriptの恩恵を存分に受けたいという場合はReactを採用しておくのが無難**かと思います。

逆にこういった部分的に親和性に欠けるということを理解した上でVue (+ TypeScript)を採用するのはアリかと思います。
上記の例の場合はundefinedのチェックはscript側でするなど工夫することで回避できることはあります。

私個人としてはVueを使うための工夫よりも根本的な部分でのサポートが欲しいと思うことが多いのでReactを採用したいと思うことが多いです。

## 双方向データバインディング
Reactは単方向データバインディングのみのサポートなのに対し、Vueは双方向データバインディングもサポートしています。

双方向データバインディング自体は小規模なフロントエンド（開発メンバーがすぐに仕様を把握できるレベル）をサクッと作る分には便利な一方で、規模が大きくなるにつれて重荷になっていくように思います。

特にReactライクな単方向でのデータ更新に加えて、`v-model`や`emit`を使った実装等、同じことを実現するのにも複数の方法があることはコードの統一感を失くしやすいです。

~~実際Vueは2 -> 3で`.sync`による双方向データバインディングを廃止していたりしますし、~~
双方向データバインディング自体がある種の諸刃の剣のようなツールだと個人的には考えています。
  - **2022/04/01 追記)** 廃止ではなく書き方が統一されたというのが正しいとフィードバックを受けたので取り消し線を入れました。

# 個人的にReactの採用を検討してほしいと思う場面

上で述べたことの逆になりますのでサラッといきます。

## 開発規模の話
- 中規模以上のチーム開発
  - TypeScriptによる恩恵
- 単方向データバインディング

## 運用面での話
- 長期の保守、安全な運用が前提
  - 後方互換性の高さ
  - TypeScriptとの親和性の良さ
    - tscによるチェック
    - JSX、TSXの表現力の高さ


## チームメンバーの話
- フロントエンドへ関心が高い
  - 新機能については概ねVueがReactに追従している状態なので、Reactをやっておけば現状大が小を兼ねてくれる。
- Vueへのこだわりがそこまでない


## 一言でまとめると
**それなりの規模で安定的な開発・運用がしたい。**

といったときでしょうか。


# Reactで辛いと思うところ

逆にReactで辛いと思われる部分もあります。

## 関数型ベースな設計
Reactには関数型的な思想が見受けられます。
例えばJSXは実質JavaScriptの関数です。
これがプレーンなHTML/CSSからの構文的な乖離を生んでいるように見え、違和感を感じる方もいると思います。

また、関数コンポーネントで記述していると、ライフサイクル等の実装をしようと思った時に全て`useEffect`という一見すると直感的ではない関数を使用することになります。
これも初見では何が起きているのか分かりにくく感じる人は多そうです。

```tsx
function App() {
  const [count, setCount] = useState(0);

  // Vueでいうmounted
  useEffect(() => {
    console.log("App mounted!!")
  }, [])

  // ライフサイクルじゃないけどVueでいうwatch
  useEffect(() => {
    console.log("count updated!!")
  }, [count])

  return (
    <div>
      <span>{count}</span>
      <button onClick={() => setCount(count + 1)}>count</button>
    </div>
  );
}
```

あと基本的な構文も`v-if`や`v-for`のような直感的なものではありません。

```tsx
function App() {
  const [visible, setVisible] = useState<boolean>(true)
  const [numbers, setNumbers] = useState<number[]>([1, 2, 3])
  
  return (
    <div>
      {
        // v-if的なやつ
        visible && <div>Hello</div>
      }
      { // v-for的なやつ
        numbers.map(number => (<div key={number}>{number}</div>))
      }
    </div>
  );
}

```

関数型風の考え方に馴染みがない人にとっては一定の慣れが必要な箇所はどうしても出てしまう気がします。

## ライブラリの選択肢が多い
これは特にStyleや状態管理ライブラリが顕著ですが、Vueに比べると選択肢が多く、慣れていないとどれを選ぶのが良いか悩まされるかもしれません。

0ベースであれば個人的には標準ベースで公式ドキュメントを読みつつ順次に触れていくのが良いように思っています。

https://reactjs.org/docs/context.html
https://reactjs.org/docs/faq-styling.html


# 最後に
Vueの方で色々語ってしまったのでとてもバランスが悪くなってしまいました🙇‍♂️

VueはEasy、ReactはSimpleとしばしば言われます。

VueにはPHPを彷彿とさせるようなとっつきやすさがありますし、Reactには関数型に影響を受けた思想（？）を感じます。
この辺は最終的には好みも影響すると思うのですが、開発時だけでなく長期の運用や開発メンバーなど色々な視点から考えてメリットデメリットを考えられるようになりたいと思い執筆しました。

私個人の視点としてはどうしても長期運用のしやすさという観点が強くなってしまうので、結構なバイアスがかかってしまったようにも感じますが、あまりこういった部分に言及されている記事は多くないと思っているのでこれはこれでありだと自分に言い聞かせています。
一意見として捉えていただければ幸いです🙇‍♂️

それぞれ使い所を考えつつ良い感じにフロントエンド開発ができればいいですね。
