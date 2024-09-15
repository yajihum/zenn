---
title: "Orvalを使ったコードの自動生成をGitHub Actionsを通して行う"
emoji: "👺"
type: "tech"
topics: ['react', 'orval', 'tanstackquery', 'openapi', 'githubactions']
published: false
---

## はじめに

OrvalはOpenAPI定義からFE用のフェッチ関数や型を自動生成してくれるライブラリです。特にTanstack Query（React Query）やSWRなどのフェッチライブラリを使用している場合に各ライブラリにあったクライアントを生成してくれます。  

私の場合は業務でTanstack Queryを使用しており、optionsやカスタムフックまで生成してくれるので実装ファイルが少なくなり、作業を効率化できて大変楽になりました！

## CIで自動生成を行うまでの全体の流れ

自動生成を行うまでの流れは以下です。

1. Orvalを正常に生成できるようにBEのCIでリントのアクションを実行し、マージする
2. BEリポジトリのdevelopブランチでOpenAPIドキュメントが更新されたらFEのリポジトリに通知する
3. FEのリポジトリで通知を受け取る
4. BEのOpenAPIファイルを参照してOrvalを使用した自動生成を行う
5. FEのリポジトリにファイルの差分を含むPull Requestを作成する
6. 問題なければマージする

## (前提) OpenAPIファイルの用意

APIの共通の仕様書となるOpenAPIファイルを用意しておきます。例えば、BEで実装コードからOpenAPIドキュメントを自動生成している場合はBEリポジトリに存在していると思います。
BEリポジトリのどの場所にOpenAPIファイルがあるかを確認しておきます。

## 1. Orvalを正常に生成できるようにBEのCIでリントのアクションを実行し、マージする

まず1では、GitHub Actions内でorvalを実行してエラーが出ないかどうかを確認するアクションを追加します。

```yaml:.github/workflows/openapi-lint.yml
name: openapi-lint

on:
  pull_request:
    paths:
      - 'xxx/docs/openapi.yaml' ## OpenAPIファイルが存在するパス
    types: [opened, synchronize]  

jobs:
  swagger-lint:
    name: Swagger Lint
    runs-on: ubuntu-latest

    steps:
      - name: Check out code
        uses: actions/checkout@v4

      - name: Install orval
        run: npm install -g orval

    　## OpenAPIファイルを指定してorvalコマンドを実行し、吐き出したログをoval.logに記録し、Errorの文字列がある場合は強制終了する
      - name: Run orval validation
        run: |
          cd xxx
          orval --input docs/openapi.yml --output /tmp/orval-output 2>&1 | tee orval.log
          grep -q -e "Error" -e "Exception" orval.log && exit 1 || exit 0

      - name: Clean up generated files
        run: rm -rf /tmp/orval-output
```

ここで注意点として、orvalコマンドを実行してエラーが発生しても、なぜか強制終了せずにワークフローが正常終了してしまうことがあります。
なので、別ファイルにログとして記録しErrorの文字列がある場合は終了にするという結構無理矢理な方法でやっています。（他にいい方法がある場合は教えてくださると助かります🙏）

このリントチェックが通ったらdevelopにマージします。

## 2. BEリポジトリのdevelopブランチでOpenAPIドキュメントが更新されたらFEのリポジトリに通知する

このワークフローを別で作成します。

```yaml:.github/workflows/notify-frontend-swagger-update.yml
name: Notify Frontend of OpenAPI Update

on:
  push:
    branches:
      - develop
    paths:
      - 'xxx/docs/openapi.yml'

jobs:
  notify-frontend:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Frontend Workflow
        uses: peter-evans/repository-dispatch@v2
        with:
          token: ${{ secrets.FRONTEND_API_CLIENT_UPDATE_TOKEN }}
          repository: aaa/bbb ##　FEのリポジトリ
          event-type: oepnapi-updated ## イベントを受け取るラベル
```

ここでは他のリポジトリに通知を送っているため、`GITHUB_TOKEN`よりも権限が強い環境変数を設定する必要があります（`token: ${{ secrets.FRONTEND_API_CLIENT_UPDATE_TOKEN }}`の部分）
GitHub Appsなどの設定方法もあるのですが、サードパーティアクションの危険性などもある[^1]ようなので`Fine-grained personal access tokens`を使用しています。

Fine-grained personal access tokensの生成の仕方：https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens

## orvalの設定ファイルを作成する

orvalで生成するファイルやコードに対して細かい設定を行うために`orval.config.ts`を作成します。
今回は`packages/resources`をルートとしてファイルを作成します。適宜自動生成ファイルを置きたい場所などに合わせて変えてください。

```ts
import { defineConfig } from 'orval'

export default defineConfig({
  sasael: {
    input: '../../backend/pkg/schoolaffairs/docs/swagger.yaml',
    output: {
      target: './src/generated/clients',
      schemas: './src/generated/models',
      client: 'react-query',
      mode: 'tags-split',
      indexFiles: false,
      urlEncodeParameters: true,
      biome: true,
      override: {
        mutator: {
          path: './src/custom-fetch.ts',
          name: 'customFetch',
        },
        query: {
          useQuery: true,
          useMutation: false,
          useSuspenseQuery: false,
          useInfinite: false,
          usePrefetch: false,
          // NOTE: 非推奨だがqueryOptionsを使用するとhookとしてoptionが生成されるためこちらを使う
          options: {
            staleTime: 5 * 60 * 1000,
            gcTime: 10 * 60 * 1000,
            retry: false,
          },
          version: 5,
        },
      },
    },
  },
})
```


## 3~5のワークフローを作成する

以下の3~5は1つのワークフローで行えます。

> 3. FEのリポジトリで通知を受け取る
> 4. BEのOpenAPIファイルを参照してOrvalを使用した自動生成を行う
> 5. FEのリポジトリにファイルの差分を含むPull Requestを作成する

```yaml:.github/workflows/update-api-client.yml
name: Update API Client

on:
  repository_dispatch:
    types: [oepnapi-updated] ## BE側で定義したイベントのラベル

jobs:
  update-api-client:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332 # v4

      - name: Setup pnpm
        uses: pnpm/action-setup@v4

      - name: Setup node
        uses: actions/setup-node@v4
        with:
          node-version-file: '.node-version'
          cache: 'pnpm'

      - name: Check out backend repository
        uses: actions/checkout@v4
        with:
          repository: sasael-inc/sasael-backend
          ref: develop
          path: backend
          token: ${{ secrets.FRONTEND_API_CLIENT_UPDATE_TOKEN }}

      - name: Install dependencies
        run: pnpm install
        working-directory: packages/resources

      - name: Run orval
        run: pnpm orval
        working-directory: packages/resources

      - name: Close existing pull requests and delete branches
        run: |
          prs=$(gh pr list --state open --head update-api-client --json number --jq '.[].number')
          for pr in $prs; do
            gh pr close $pr --delete-branch
          done
        env:
          GH_TOKEN: ${{ secrets.FRONTEND_API_CLIENT_UPDATE_TOKEN }}

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@d121e62763d8cc35b5fb1710e887d6e69a52d3a4 # v7
        with:
          token: ${{ secrets.FRONTEND_API_CLIENT_UPDATE_TOKEN }}
          commit-message: 'update API client based on new swagger.yaml'
          base: develop
          branch: update-api-client
          delete-branch: true
          title: 'update API client'
          body: 'バックエンドのSwagger定義ファイルが更新されたため、最新のAPIクライアントを作成しました。'
          committer: 'github-actions[bot] <github-actions[bot]@users.noreply.github.com>'
```

- pnpmの部分は適宜使用しているパッケージマネージャーに変えてください。
- working-directoryはorvalの設定

[^1]: https://zenn.dev/tmknom/articles/github-apps-token#%E3%82%B5%E3%83%BC%E3%83%89%E3%83%91%E3%83%BC%E3%83%86%E3%82%A3%E3%82%A2%E3%82%AF%E3%82%B7%E3%83%A7%E3%83%B3%E3%81%AB%E3%82%88%E3%82%8Bgithub-apps%E3%83%88%E3%83%BC%E3%82%AF%E3%83%B3%E3%81%AE%E7%94%9F%E6%88%90