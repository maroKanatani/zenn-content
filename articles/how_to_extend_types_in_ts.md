---
title: "既存のTypeScriptの型を拡張する際に考えたこと"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript"]
published: true
---

TypeScriptで開発をする際、外部ライブラリをラップするなどして保守性を高めようとすることがあります。
その際型の整合性をとるために考えたことを書き出してみます。

例としてユーザーデータに対して定義されたActionを拡張するような場合を考えてみます。
以下を既存の型とします。

```ts
type User = { id: number, name: string, role: "Admin" | "User" }
type Action = 
  { method: "GET" } | 
  { method: "POST", body: User } | 
  { method: "PATCH", body: Partial<Omit<User, "id">>, id: User["id"] } | 
  { method: "DELETE", id: User["id"] }

// ex.
const actionList: Action[] = [
    { method: "GET" },
    { method: "POST", body: { id: 2, name: "Bob", role: "User" }},
    { method: "PATCH", body: { role: "Admin" }, id: 3 },
    { method: "DELETE", id: 1 }
]
```

この型をベースに、例えばGETには`read`権限、POST以上は`write`権限が必要といったように、
methodとそれに関わるパラメータ以外に権限という概念を追加しようとしてみます。

2パターンの方法を紹介します。

# 既存の型にプロパティを追加する

こちらはシンプルに`&`を使って既存の型を拡張するための型を定義するようなパターンです。

```ts
// 既存の型にrequiredプロパティを追加する型
type WithPermission<T> = T & {
    required: "read" | "write"
}

const actionList: WithPermission<Action>[] = [
    { method: "GET", required: "read" },
    { method: "POST", body: { id: 2, name: "Bob", role: "User" }, required: "write" },
    { method: "PATCH", body: { role: "Admin"}, id: 3, required: "write" },
    { method: "DELETE", id: 1, required: "write" }
]
```

この場合、例えばOmit等を使って追加したプロパティを除き、元の型を返したりしようと思うと大変な場合があります。
以下のように`{ required: "write" }`を持つオブジェクトのみをフィルタする汎用的な関数`filterWrite`を定義し、
`WithPermission<Action>`のリストをフィルタしてみます。

```ts
const actionList: WithPermission<Action>[] = [
    { method: "GET", required: "read" },
    { method: "POST", body: { id: 2, name: "Bob", role: "User" }, required: "write" },
    { method: "PATCH", body: { role: "Admin" }, id: 3, required: "write" },
    { method: "DELETE", id: 1, required: "write" }
]

// requiredプロパティがwriteのものだけを返す汎用的な関数
const filterWrite = <T extends WithPermission<Record<string, unknown>>>(items: T[]) => {
    return items.flatMap(({ required, ...item }) => {
        return required === "write" ? [item] : []
    })
}

// writeActionsの型はOmit<WithPermission<Action>, "required">[]となる
const writeActions = filterWrite(actionList)

writeActions.map(action => {
    // actionは { method: "GET" | "POST" | "PATCH" | "DELETE"; } 型になる。Action型にならない。
})
```

`filterWrite`が返す値は`Omit<T, "required">[]`となります。
上の例ではAction型がUnion型として定義されているので、それに対してOmitを使用すると共通しているプロパティのみの型になってしまいます。
この場面では本当は`WithPermission`に渡す前のAction型として扱って欲しいです。

これを解消しようと思うと、例えば以下のように`as`を使う方法があります。

```ts
const filterWrite = <T extends Record<string, unknown>>(actions: WithPermission<T>[]) => {
    return actions.flatMap(({ required, ...action }) => {
        return required === "write" ? [action as T] : []
    })
}

// writeActionsの型はAction[]となる
const writeActions = filterWrite(actionList)
```

`as`を使いたくない場合は以下の記事のようにUnion型のそれぞれに対してOmitを適用するような型を自分で用意すれば可能かもしれません。
https://zenn.dev/kimitsu/articles/48dc59129c5569


# 既存の型を保持する型を定義する

`as`はコンパイラの型情報を上書きするような挙動となりますし、
自前で型を用意してまで型パズルを頑張りたくないということもあるかと思うので
別のアプローチも考えてみました。以下のように既存の型を保持する別の型を定義するパターンです。

```ts
// 既存の型Tをitemに格納し、別途requiredプロパティを保持。
type WithPermission<T> = {
    item: T,
    required?: "read" | "write"
}

const actionList: WithPermission<Action>[] = [
    { item: { method: "GET" }, required: "read" },
    { item: { method: "POST", body: { id: 2, name: "Bob", role: "User" }}, required: "write" },
    { item: { method: "PATCH", body: { role: "Admin" }, id: 3 }, required: "write" },
    { item: { method: "DELETE", id: 1 }, required: "write" }
]
```

データ構造が前者のものと異なるのと、オブジェクトがネストされたときは別途考慮が必要ですが、
以下のように`as`も自前の型も不要になるので、要件的に問題なければこの方法がハマることもありそうです。

```ts
const filterWrite = <T extends Record<string, unknown>>(actions: WithPermission<T>[]) => {
    return actions.flatMap(({ required, item }) => {
        return required === "write" ? [item] : []
    })
}

// Action[]型
const writeActions = filterWrite(actionList)
```


# まとめ
既存の型を拡張する方法について自分の考えたことを書いてみました。
他にも色々なアプローチがあるかもしれません。

いずれにせよTypeScriptを使って開発を進める場合、多かれ少なかれ何かしらのライブラリを使うことが多いと思いますし、
ある程度の規模になるとライブラリのラップやユーティリティを用意する必要も出てくると思います。

こういったときに扱う型が単なるオブジェクト型なのかUnion型なのかは意識しないといけないですね。
