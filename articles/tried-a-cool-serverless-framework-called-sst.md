---
title: "sstといういけてるサーバーレスフレームワークを使ってみた"
emoji: "😸"
type: "tech"
topics: ["sst", "serverless"]
published: true
---
# sstとは
https://sst.dev/chapters/what-is-sst.html
AWS用のサーバーレスフレームワークで、ローカルでの開発やデプロイを構築してくれる。
serverless frameworkという有名なフレームワークがあるが、それと似ている。
筆者はserverless frameworkを使ったことがないため比較はできないが、sstは割と最近(2021年)に出ており、serverless frameworkの使いにくい点が色々解消されているようだ。

# AWSのセットアップ
sstを利用するには、AWSアカウントが必要なので、作る。
また、awsのアクセスキーが必要なので、[こちらの手順](https://sst.dev/chapters/create-an-iam-user.html)を元に作ろうと思ったが、アクセスキーを作るときに、代替案を検討しろと言われた。
どうやらアクセスキーには有効期限もなくセキュリティリスクが高いらしい。わたしは心配性なので、素直に警告に従うことにした。
代替案としてはcloud shellを使うか、IAM Identity Centerを使うかだ。
localでcliを使いたかったので、IAM Identity Center使うことにした。
設定方法は他にわかりやすい日本語の記事があるので割愛する。

# サーバー立ち上げ
:::message
このライブラリは頻繁にアップデートされてるみたいなので、以下の手順で動くかは保証できない
:::

templateを元にひな形を作成できるみたいなので、作成する。
templateは[ここ](https://github.com/serverless-stack/sst/tree/master/packages/create-sst/bin/presets/)から選べるらしい。
指定方法はpresets/以下のパスを指定すればいいみたい。
graphqlとdynamoを使ってみたいので、以下を実行。
`npx create-sst@latest sst-sample --template=graphql/dynamo`

作成したディレクトリに移動
`cd sst-sample`

リージョンを東京に変更しておく

```diff typescript:sst-config.ts
import { SSTConfig } from "sst";
import { Api } from "./stacks/Api";
import { Web } from "./stacks/Web";
import { Database } from "./stacks/Database";

export default {
  config(_input) {
    return {
      name: "sst-sample",
-       region: "us-east-1",
+       region: "ap-northeast-1",
    };
  },
  stacks(app) {
    app
      .stack(Database)
      .stack(Api)
      .stack(Web);
  }
} satisfies SSTConfig;
```

そして立ち上げ
`npm i`
`npx sst dev --profile dev`
  profileはaws configureで設定したものを指定する。

```
➜  App:     sst-sample
   Stage:   dev
   Console: https://console.sst.dev/sst-sample/dev

|  Database db/Table AWS::DynamoDB::Table CREATE_COMPLETE 
|  Database CustomResourceHandler/ServiceRole AWS::IAM::Role CREATE_COMPLETE 
|  Database db/Parameter_tableName AWS::SSM::Parameter CREATE_COMPLETE 
|  Database CustomResourceHandler AWS::Lambda::Function CREATE_COMPLETE 
|  Database AWS::CloudFormation::Stack CREATE_COMPLETE 
|  Api api/Api AWS::ApiGatewayV2::Api CREATE_COMPLETE 
|  Api api/LogGroup AWS::Logs::LogGroup CREATE_COMPLETE 
|  Api api/Api/DefaultStage AWS::ApiGatewayV2::Stage CREATE_COMPLETE 
|  Api api/Parameter_url AWS::SSM::Parameter CREATE_COMPLETE 
|  Api api/Lambda_POST_--graphql/ServiceRole AWS::IAM::Role CREATE_COMPLETE 
|  Api CustomResourceHandler/ServiceRole AWS::IAM::Role CREATE_COMPLETE 
|  Api CustomResourceHandler AWS::Lambda::Function CREATE_COMPLETE 
|  Api api/Lambda_POST_--graphql/ServiceRole/DefaultPolicy AWS::IAM::Policy CREATE_COMPLETE 
|  Api api/Lambda_POST_--graphql AWS::Lambda::Function CREATE_COMPLETE 
|  Api api/Route_POST_--graphql/Integration_POST_--graphql AWS::ApiGatewayV2::Integration CREATE_COMPLETE 
|  Api api/Lambda_POST_--graphql/EventInvokeConfig AWS::Lambda::EventInvokeConfig CREATE_COMPLETE 
|  Api api/Route_POST_--graphql AWS::ApiGatewayV2::Route CREATE_COMPLETE 
|  Api api/Route_POST_--graphql AWS::Lambda::Permission CREATE_COMPLETE 
|  Api AWS::CloudFormation::Stack CREATE_COMPLETE 
✔  Pothos: Extracted pothos schema
|  Web site/Parameter_url AWS::SSM::Parameter CREATE_COMPLETE 
|  Web CustomResourceHandler/ServiceRole AWS::IAM::Role CREATE_COMPLETE 
|  Web CustomResourceHandler AWS::Lambda::Function CREATE_COMPLETE 
|  Web AWS::CloudFormation::Stack CREATE_COMPLETE 

✔  Deployed:
   Database
   Api
   API: https:***
   Web
```

たったこれだけでapi gateway, lambda, dynamoなどが展開された。
urlはマスクしているが、localではなく実際のapi gatewayのURLになっている。


# フロントエンド立ち上げ
`cd packages/web`
`npm run dev -- --profile dev`

reactアプリがlocalhost:3000で立ち上がる
```
  vite v2.9.15 dev server running at:

  > Local: http://localhost:3000/
  > Network: use `--host` to expose

  ready in 196ms.
```

しかしなんかエラーでたので、直す
```diff tsx:packages/web/src/main.tsx
import React from "react";
import ReactDOM from "react-dom/client";
import {
- Create
+ createClient,
  cacheExchange,
  fetchExchange,
  Provider as UrqlProvider,
} from "urql";
import { BrowserRouter, Routes, Route, Navigate } from "react-router-dom";
import Home from "./pages/Home";
import Article from "./pages/Article";
import "./globals.css";

const urql = createClient({
  url: import.meta.env.VITE_GRAPHQL_URL,
  exchanges: [cacheExchange, fetchExchange],
});

...
```

画面が表示された。
![トップ画面](/images/tried-a-cool-serverless-framework-called-sst/sst-sample-top-page.png)

# 動作確認
## レコードを登録してみる
title, urlを入力してsubmitすると、dynamoに登録される。
![dynamoレコード](/images/tried-a-cool-serverless-framework-called-sst/dynamo-record.png)

## コードを変更してみる
Articleにtypeというプロパティを追加してみた。
```diff typescript:packages/core/src/article.ts
...

export const ArticleEntity = new Entity(
  {
    model: {
      version: "1",
      entity: "Article",
      service: "scratch",
    },
    attributes: {
      articleID: {
        type: "string",
        required: true,
        readOnly: true,
      },
      title: {
        type: "string",
        required: true,
      },
      url: {
        type: "string",
        required: true,
      },
+      type: {
+        type: "string",
+      }
    },
    indexes: {
      primary: {
        pk: {
          field: "pk",
          composite: [],
        },
        sk: {
          field: "sk",
          composite: ["articleID"],
        },
      },
    },
  },
  Dynamo.Configuration
);

export type ArticleEntityType = EntityItem<typeof ArticleEntity>;

- export async function create(title: string, url: string) {
+ export async function create(title: string, url: string, type: string) {
  const result = await ArticleEntity.create({
    articleID: ulid(),
    title,
    url,
    type,
  }).go();

  return result.data;
}

...
```
```diff typescript:packages/functions/src/graphql/types/article.ts
import { Article } from "@sst-sample/core/article";
import { builder } from "../builder";

const ArticleType = builder
  .objectRef<Article.ArticleEntityType>("Article")
  .implement({
    fields: (t) => ({
      id: t.exposeID("articleID"),
      url: t.exposeString("url"),
      title: t.exposeString("title"),
+      type: t.exposeString("type", { nullable: true }),
    }),
  });

builder.queryFields((t) => ({
  article: t.field({
    type: ArticleType,
    args: {
      articleID: t.arg.string({ required: true }),
    },
    resolve: async (_, args) => {
      const result = await Article.get(args.articleID);

      if (!result) {
        throw new Error("Article not found");
      }

      return result;
    },
  }),
  articles: t.field({
    type: [ArticleType],
    resolve: () => Article.list(),
  }),
}));

builder.mutationFields((t) => ({
  createArticle: t.field({
    type: ArticleType,
    args: {
      title: t.arg.string({ required: true }),
      url: t.arg.string({ required: true }),
+      type: t.arg.string({ required: true }),
    },
-    resolve: (_, args) => Article.create(args.title, args.url),
+    resolve: (_, args) => Article.create(args.title, args.url, args.type),
  }),
}));
```
frontendもよしなに修正し、submitすると、見事に追加された。
![typeプロパティを追加](/images/tried-a-cool-serverless-framework-called-sst/add-type-property.png)

# 感想

## スムーズな開発体験
とても簡単にサーバーレス環境をデプロイできた。
それだけでなく、localでの開発もスムーズだった。特にバックエンドの修正がすぐ反映されて動作確認できるのは素晴らしい。

この辺は独自の仕組みを使ってるようで、[こちら](https://docs.sst.dev/live-lambda-development)に詳しく説明がある。
ざっくりいうと、
`client => AWS上のAPI Gateway => AWS上のlambda => localのfunction => AWS上のlambda => client`
という流れでリクエストが処理されるようだ。こうすることでlocalにapi gatewayやlambdaに似た環境を用意せずとも、開発ができる。

上記の仕組みのため、vscodeでデバッグもできるので、開発が捗りそう。

## 保守性も重視
[Learn](https://docs.sst.dev/learn/domain-driven-design)でDDDが紹介されていた。
上で作成したtemplateもDDDを意識した構成になっているようで、そもそもmonorepoになっており、
- packages/core/
  ドメインモデルやサービス、レポジトリなど
- packages/function
  graphqlのスキーマ・リゾルバの定義など

といった形でパッケージで分かれてるので、lambda, graphqlといったプレゼンテーション層を意識せずにビジネスロジックを書くことができる構成になっている。
サーバーレスだけど統率の取れたコードを書けそうなので、個人開発だけでなくスタートアップでも使えそう。

## まだ情報が少ない
sstで検索してもSST（ソーシャルスキルトレーニング）が先に出てきてしまうなど、まだまだ知名度は低そうだが、star数は13.7kとそこそこ高い。
ライブラリのアップデートやdiscordも活発なので、今後も個人開発などで使ってみたい。