---
title: "Page Object Modelでフロントエンドのテストを書いてみる"
emoji: "🎭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["test", "playwright", "frontend"]
published: false
---

Playwrightのドキュメントを眺めていると`Page Object Model`という実装パターンがあり、これが便利そうだと思ったので愚直に実装したパターンとじっくり見比べてみたいと思いました。

https://playwright.dev/docs/test-pom

# 私の環境
Node 18.5.x
M1 Mac
@playwright/test 1.25.1


# テスト対象のアプリ
今回は`frourio`を使ってサンプルアプリを用意しました。

https://frourio.com/

`frourio`には`create-frourio-app`という雛形を作成するコマンドが用意されており、これを叩くだけでTODOアプリが作成されます。
バックエンドもフロントエンドも用意してくれます。

https://frourio.com/docs/reference/cfa/gui

```sh
$ npx create-frourio-app
```

今回は全てデフォルト設定で作成しました。

しばらくしてから`localhost:8000`にアクセスすると以下のようなTODOアプリが表示されます。
画面中央にあるテキストボックスに入力後、ADDボタンを押すことでデータが追加されます。
![](/images/playwright_with_page_object_model/todo-app.png =700x)

# 下準備
テストしやすくするために数カ所`aria-label`を付与しておきます。

```diff tsx:src/pages/index.tsx（抜粋）

const Home: NextPage = () => {
  ...
  return (
    <Layout>
      ...
      <div>
        <form style={{ textAlign: 'center' }} onSubmit={createTask}>
+         {/** todo-input **/}
          <input
            value={label}
+           aria-label={'todo-input'}
            type="text"
            onChange={inputLabel}
          />
+          {/** todo-add-button **/}
+          <input type="submit" aria-label={'todo-add-button'} value="ADD" />
        </form>
+        {/** todo-list **/}
+        <ul className={styles.tasks} aria-label={'todo-list'}>
          {tasks.map((task) => (
            ...
          ))}
        </ul>
      </div>
    </Layout>
  )
}
```

# まずは愚直に実装してみる
愚直に実装すると以下のようになるかと思います。
（本来はテストケースを分けたりすることが望ましいですが、今回は操作の流れをわかりやすくしたいので1ケースで何度もassertしています。）

```ts:tests/non-pom-pattern.spec.ts
import { test, expect } from '@playwright/test'

test('愚直に実装してみたパターン', async ({ page }) => {
  // 画面遷移
  await page.goto('http://localhost:8000')

  // 初期表示ではTODO Listが0件であるのを確認する
  const todoListArea = page.locator('[aria-label="todo-list"]')
  const todoList = todoListArea.locator('li')
  await expect(todoList).toHaveCount(0)

  // TODOを追加する
  const todoInput = page.locator('[aria-label="todo-input"]')
  await todoInput.fill('TODO1')
  const addTodoButton = page.locator('[aria-label="todo-add-button"]')
  await addTodoButton.click()
  // 1件追加され、追加したデータはチェックされていないこと
  await expect(todoList).toHaveCount(1)
  await expect(todoList.first()).toHaveText('TODO1')
  await expect(todoList.first().locator('role=checkbox')).not.toBeChecked()

  // チェックをつける
  await todoList.first().locator('role=checkbox').click()
  // チェックされること
  await expect(todoList.first().locator('role=checkbox')).toBeChecked()

  // TODOを削除する
  await todoList
    .first()
    .locator('role=button', {
      hasText: 'DELETE'
    })
    .click()
  // 0件になること
  await expect(todoList).toHaveCount(0)
})
```

:::message
データ初期化の処理を書いてないので、再実行する場合は作成してしまったTODOをDELETEしてから実行するようにしてください。
:::

今回はあまり大きな規模のアプリケーションではないのでそこまで困らないのですが、テストコード内にlocator指定がしばしば見られ、DOMの構造を意識したコードになってしまっています。
また、操作が共通化されていないので、同じ操作を繰り返したりする必要が出てきた際に何度も似たような実装を書く必要があります。

# Page Object Modelで実装してみる

まずは今回のTODOアプリの画面を表すPage Objectを作成します。

```ts:pageObject/todoPage.ts
import { Locator, Page } from '@playwright/test'

export class TodoPage {
  readonly page: Page
  readonly todoListArea: Locator
  readonly todoList: Locator
  readonly addTodoInput: Locator
  readonly addTodoButton: Locator

  constructor(page: Page) {
    this.page = page
    this.todoListArea = page.locator('[aria-label="todo-list"]')
    this.todoList = this.todoListArea.locator('li')
    this.addTodoInput = page.locator('[aria-label="todo-input"]')
    this.addTodoButton = page.locator('[aria-label="todo-add-button"]')
  }

  // 対象のページへ遷移
  async goto() {
    await this.page.goto('http://localhost:8000')
  }

  // TODOを追加する操作
  async addTodo(title: string) {
    await this.addTodoInput.fill(title)
    await this.addTodoButton.click()
  }

  // index番目のTODOのチェックボックスを取得する
  getTodoCheckbox(index: number): Locator {
    return this.todoList.nth(index).locator('role=checkbox')
  }

  // index番目のTODOのチェックボックスをクリックする操作
  async checkTodo(index: number) {
    await this.todoList.nth(index).locator('role=checkbox').click()
  }

  // index番目のTODOを削除する操作
  async deleteTodo(index: number) {
    await this.todoList
      .nth(index)
      .locator('role=button', {
        hasText: 'DELETE'
      })
      .click()
  }
}
```

テスト中に行いたいクリックや入力等の操作をメソッドとしてPage Objectに定義しておき、
操作の中で必要なセレクタの指定等、DOMを意識するようなものは基本的にこの中に閉じ込めておきます。

**これによりテストコードは操作とassertionのみ注視しておけばよく、DOMの構造に変化があった際にも修正箇所を最小限に留めることができます。**

テストコードは以下のようになります。
とても見通しが良くなったように見えます。

```ts:pom-pattern.spec.ts
import { test, expect } from '@playwright/test'
import { TodoPage } from './pageObject/todoPage'

test('Page Object Modelでの実装', async ({ page }) => {
  const todoPage = new TodoPage(page)
  // 画面遷移
  await todoPage.goto()
  // 初期表示ではTODO Listが0件であるのを確認する
  await expect(todoPage.todoList).toHaveCount(0)

  // TODOを追加する
  await todoPage.addTodo('TODO1')
  // 1件追加され、追加したデータはチェックされていないこと
  await expect(todoPage.todoList).toHaveCount(1)
  await expect(todoPage.todoList.first()).toHaveText('TODO1')
  await expect(todoPage.getTodoCheckbox(0)).not.toBeChecked()

  // チェックをつける
  await todoPage.checkTodo(0)
  // チェックされること
  await expect(todoPage.getTodoCheckbox(0)).toBeChecked()

  // TODOを削除する
  await todoPage.deleteTodo(0)
  // 0件になること
  await expect(todoPage.todoList).toHaveCount(0)
})
```

# 最後に
Page Object Modelでテストを書いてみました。
バックエンドと比較してフロントエンドの場合、UIの変更に伴ってすぐに壊れてしまう部分が多いです。
（UIはユーザーのフィードバックを受けて仕様変更や機能の追加が入りやすい部分でもあると私は考えています。）

プロダクトコードはもちろんのこと、テストの方にも同じくらい力を入れてメンテしやすいコードを書いていきたいと改めて感じました。
