---
title: "自分専用の複数LLM対応チャットボットを作った"
emoji: "👏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["JavaScript", "TypeScript", "aisdk", "cloudrun"]
published: false
---

複数LLMを切り替えて使えるチャットボット(ChatGPTクローン)はあまりない気がしていて、あっても自前のapiKeyを使えないものがほとんどかと思う。
自前のapiKeyを使えたとして、知らないサービスにそれを預けるのは気が引ける。
ということで、自分専用のチャットボットを作った。
localだけで動けばいいのであれば、[LibreChat](https://github.com/danny-avila/LibreChat) というOSSが完成度高くてよかったが、
使ってるうちにやはり複数端末でやりたいとなってきて、いちいち環境構築するのは面倒なので、ホスティングしたくなった。
LibreChatは機能もりもりでホスティングするのが少し面倒そうなので、一から自前で作ることにした。
ちなみに最短で実現したいなら、https://github.com/vercel/ai-chatbot をvercelにデプロイするのが良さそう。

# Demo
![](/images/create-chat-bot-with-multiple-llm/llm-chat-demo.gif)


# 使用した技術

### TypeScript
TypeScriptしか書けないので。
後述するvercel ai sdkがとても便利で、TypeScript使っててよかったと思っている。
webフレームワークはReact Routerを採用。

### Cloud Run
ホスティング先は使い慣れているcloud runにした。最小インスタンス0で月額コストは数円程度。
後述するIAPを設定すれば、自分だけにアクセスを制限するのを、コードを書かずに実現できる。
cloudflare workersを検討したが、アクセスが香港経由になってopenaiなどに弾かれるというのを見たのでやめた。

### Supabase
DBは無料で使いたく、supabaseの2プロジェクト制限でやりくりしている。
今回認証は不要なので、シンプルにDBに直接アクセスして使っている。
ORMはdrizzleを使用している。
dbのmigrationはsupabaseの方で管理している。というのも、本番のDBにもlocalから直接pushしており、supabase db pushコマンドを使いたいから。
localのsupabaseのGUIでスキーマをいじる -> drizzle pull という方法で同期している。
今の所問題は起きてない。

### vercel ai sdk
これが非常に便利で、複数LLMへのリクエスト部分を抽象化してくれるし、streamingとかを自前で実装せず、backendからfrontendまで全部やってくれる。
複数LLM対応するにあたり最初LangChainとか使ったほうがいいのかと思ったが、そこまで大袈裟なものを使いたくなかったので、助かった。
ドキュメントも充実しており、だいたいの問題は解決した。

### その他UIとか
あまりこだわりはないので、claudeに進められるがままにshadcnとtailwindを使用。
markdownのレンダリングはreact-markdown と react-syntax-highlighterを使用。

# ハマったところなど

## Cloud Run に自分だけアクセスできるようにする
Cloud Run に IAP というものを設定すると、自分のgoogleアカウントでログインした場合のみアクセスできるよう、制限できる。
https://cloud.google.com/run/docs/securing/identity-aware-proxy-cloud-run?hl=ja 
Cloud Run へのアクセスをIAPがキャッチして、認証してくれるみたい。これはCloud Run作成時に自動で作られる ~.run.app というURLにも適用される。
ただこのIAPを有効化するには、組織というのが必要になる。組織を作るには、Cloud Identity や Google Workspaceに登録する必要があるみたいで、個人のgoogleアカウントで作ってるGoogle Cloudプロジェクトだとダメだった。
Cloud Identityは無料で使えるが、ドメインを登録する必要があるので、その費用はかかる。
以下のブログがわかりやすかった。
https://blog.g-gen.co.jp/entry/google-cloud-organization-explained

## チャットの新規作成からstreaming開始までの流れ
チャットの新規作成画面 -> streaming開始

## react markdownのレンダリングが重い

## 自分のメッセージを送信したらそれを一番上に表示するやつ
