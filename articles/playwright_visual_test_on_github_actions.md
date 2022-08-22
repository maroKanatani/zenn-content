---
title: "PlaywrightでのVisual Regression TestをGithub ActionsでCIする"
emoji: "📸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["test", "playwright", "docker", "githubactions", "frontend"]
published: true
---

Playwrightでのテストが結構快適だと思っているものの、Visual Regression Testを実施した際にハマる部分があったのでまとめてみます。

# 最終的に出来上がったもの
https://github.com/Kanatani28/playwright-vrt

# 私の環境
- M1 Mac
- Node 18.5

# プロジェクトを作成
[公式ドキュメント](https://playwright.dev/docs/intro)に従って以下のような設定で適当なプロジェクトを作成します。  
今回はGithub Actionsで動かしてみるので、CLIでworkflowのフォルダも一緒に作成しています。

```sh
# playwright-vrtという名前で作成
$ npm init playwright@latest playwright-vrt

Getting started with writing end-to-end tests with Playwright:
Initializing project in 'playwright-vrt'
✔ Do you want to use TypeScript or JavaScript? · TypeScript
✔ Where to put your end-to-end tests? · tests
✔ Add a GitHub Actions workflow? (y/N) · true
```

動作確認

```sh
$ cd playwright-vrt
$ npx playwright test
Running 3 tests using 3 workers

  3 passed (2s)

To open last HTML report run:

  npx playwright show-report
```

# Github Actionsで動かしてみる
`.github/workflows/playwright.yml`を少し編集します。  
各種actionのバージョンを`v2 -> v3`に上げているのと、
Actionsのトリガーに`workflow_dispatch`を追加して手動実行できるようにしています。

```yml:.github/workflows/playwright.yml
name: Playwright Tests
on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:
jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: '14.x'
    - name: Install dependencies
      run: npm ci
    - name: Install Playwright Browsers
      run: npx playwright install --with-deps
    - name: Run Playwright tests
      run: npx playwright test
    - uses: actions/upload-artifact@v3
      if: always()
      with:
        name: playwright-report
        path: playwright-report/
        retention-days: 30
```

この状態でGithubにpushするとPlaywrightのテストがGithub Actionsで実行されます。

# Visual Regression Testしてみる

[公式](https://playwright.dev/docs/test-snapshots)を参考にVRTしてみます。
最後の行に追記してVRTしています。

```ts
import { test, expect } from '@playwright/test';

test('homepage has Playwright in title and get started link linking to the intro page', async ({ page }) => {
  await page.goto('https://playwright.dev/');

  // Expect a title "to contain" a substring.
  await expect(page).toHaveTitle(/Playwright/);

  // create a locator
  const getStarted = page.locator('text=Get Started');

  // Expect an attribute "to be strictly equal" to the value.
  await expect(getStarted).toHaveAttribute('href', '/docs/intro');

  // Click the get started link.
  await getStarted.click();

  // Expects the URL to contain intro.
  await expect(page).toHaveURL(/.*intro/);

  // VRT実行部分
  await expect(page).toHaveScreenshot("landing.png");
});
```

追記できたら実行してみます。  
初回は画像データが無いので`--update-snapshots`オプションを付けて実行することで画像データが用意されます。

```sh
$ npx playwright test --update-snapshots
```

`tests`ディレクトリ下に画面キャプチャが用意されます。  
これ以降`--update-snapshots`をつけずに実行することで、VRTが実施できます。

```sh
# ↑で作成した画像と実施結果を比較する
$ npx playwright test
```

# Github Actionsで動かす

ここまでローカルでのVRTができましたが、CIで回そうとすると失敗してしまうことがあります。  
今回であればローカルで実行した際に取得される画像はMacで取得した画像ということになります。
**Playwrightは画面キャプチャをOSプラットフォーム毎に作成する**ため、CIで動かしているLinuxで取得した画像が用意されてないというエラーが出てきます。

```:エラー例（landing-chromium-linux.pngというファイルが存在しない）
Running 3 tests using 1 worker
××F××F××F

  1) [chromium] › example.spec.ts:3:1 › homepage has Playwright in title and get started link linking to the intro page 

    Error: /home/runner/work/playwright-vrt/playwright-vrt/tests/example.spec.ts-snapshots/landing-chromium-linux.png is missing in snapshots.

      20 |
      21 |   // VRT実行部分
    > 22 |   await expect(page).toHaveScreenshot("landing.png");
         |                      ^
      23 | });
      24 |

        at /home/runner/work/playwright-vrt/playwright-vrt/tests/example.spec.ts:22:22
```

なのでLinuxでのキャプチャを事前に用意する必要があります。  
今回はDockerを使って取得してみます。

# DockerでのLinux用キャプチャ取得

以下のような`Dockerfile`, `docker-compose.yml`, `.dockerignore`を用意します。

```Dockerfile:Dockerfile
FROM mcr.microsoft.com/playwright:v1.25.0-focal

WORKDIR /work

COPY package*.json ./

RUN npm ci

CMD ["npx", "playwright", "test"]
```

```yml:docker-compose.yml
version: '3'
services:
  playwright:
    build:
      context: .
    working_dir: /work
    volumes:
      - ./:/work
    network_mode: host
    command: npx playwright test
```

```:.dockerignore
node_modules
```

ビルド&実行してみます。

```sh
$ docker-compose build
$ docker-compose run playwright npx playwright test --update-snapshots
```

これでLinuxでのキャプチャが取得できます。


あとはPushしたらいい感じにCIされるはず・・・


と思いきや、まだこの状態では失敗してしまいます。
artifactをダウンロードしてから確認すると以下のように差分が出てきました。

![](/images/playwright_visual_test_on_github_actions/actual.png =600x)
*Github Actions上で取得されたもの*


![](/images/playwright_visual_test_on_github_actions/expected.png =600x)
*ローカルのDockerで取得されたもの*

Github ActionsのUbuntuの環境とPlaywrightのDocker Imageをベースにした環境でどうもフォント等が異なるようです。

とりあえずローカルと同じコンテナで実行がしたいと思ったので[こちらの記事](https://zenn.dev/aokiken/articles/b6e9c0eb73ed56#github-actions%E3%81%A7%E5%AE%9F%E8%A1%8C%E3%81%99%E3%82%8B)と同じようにimageを指定して実行してみます。

```diff yml:.github/workflows/playwright.yml
name: Playwright Tests
on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:
jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
+    container:
+      image: mcr.microsoft.com/playwright:v1.25.0-focal
    steps:...
```


# 今度はFirefoxのエラーが・・・
ここまでで実行してみると今度は以下のようにFirefoxのエラーが出ていました。

```
  1) [firefox] › example.spec.ts:3:1 › homepage has Playwright in title and get started link linking to the intro page 

    browserType.launch: Browser.enable): Browser closed.
    ==================== Browser output: ====================
    <launching> /ms-playwright/firefox-1344/firefox/firefox -no-remote -headless -profile /tmp/playwright_firefoxdev_profile-JAgBGP -juggler-pipe -silent
    <launched> pid=832
    [pid=832][err] Running Nightly as root in a regular user's session is not supported.  ($HOME is /github/home which is owned by uid 1001.)
    [pid=832] <process did exit: exitCode=1, signal=null>
    [pid=832] starting temporary directories cleanup
    =========================== logs ===========================
    <launching> /ms-playwright/firefox-1344/firefox/firefox -no-remote -headless -profile /tmp/playwright_firefoxdev_profile-JAgBGP -juggler-pipe -silent
    <launched> pid=832
    [pid=832][err] Running Nightly as root in a regular user's session is not supported.  ($HOME is /github/home which is owned by uid 1001.)
    [pid=832] <process did exit: exitCode=1, signal=null>
    [pid=832] starting temporary directories cleanup
    ============================================================
```




詳しいことはあまりわかりませんでしたが、playwrightのIssueを確認してみると一件それらしきものがヒットしました。

https://github.com/microsoft/playwright/issues/6500

Github側に問題があって私達には解決できない問題とのことで、`$HOME=/root`を指定してコマンドを実行することで回避できるそうです。
これを付与してとりあえずは各種ブラウザでテストが成功していることを確認できました。

```diff yml:.github/workflows/playwright.yml
@@ -21,7 +21,7 @@ jobs:
    - name: Install Playwright Browsers
      run: npx playwright install --with-deps
    - name: Run Playwright tests
-      run: npx playwright test
+      run: HOME=/root npx playwright test
    - uses: actions/upload-artifact@v3
      if: always()
      with:
```

# 所感
Playwrightを使ってVRTをしてみました。
VRTだとDOMの構造に囚われないテストができる反面、フォントなど、VRT以外のテストではそこまで気にしなかった部分でハマりました。
上手く取り入れることでフロントエンド修正時の影響範囲をより把握しやすくなるかもしれません。

昨今はフロントエンドでもPlaywrightを始め、Storybookやmswなど、便利なツールがどんどん出てきてテストが書きやすくなってきた印象があります。
フロントエンドでもテストをガンガン書いていきましょう！