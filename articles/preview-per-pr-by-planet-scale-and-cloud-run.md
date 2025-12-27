---
title: "PlanetScaleとCloud RunでPRごとのpreview環境を作る"
emoji: "🐷"
type: "tech"
topics: ["planetscale", "cloudrun", "TypeScript"]
published: false
---

# 背景
claude code on webを使いながら開発していると、動作確認にわざわざPCで開発サーバーを立ち上げるのは面倒だ。
そこで、claudeがPRを作成したら、preview用の環境をデプロイして動作確認だけしたいと考えた。
小規模なwebアプリならDBとwebサーバーくらいがPRごとに独立してくれればよく、比較的簡単に導入できる。
この記事ではCloud Runのタグ付きリビジョンと、PlanetSclaeのブランチ機能を使って、PRごとに独立したpreview環境を構築する方法を紹介する。

# できたもの
PR作成 -> github actions起動 -> PlanetScaleのbranchを作成 -> drizzleでschemaをpush -> Cloud Runをtag付きでデプロイ -> botがURLをPRにコメント
![](/images/preview-per-pr-by-planet-scale-and-cloud-run/demo-pr-comment.png)

# 主要技術スタックと選定理由

### PlanetScale
改悪してから全くwatchしてなかったが、最近postgresが使えるようになった。
また、最安で1DBにつき5$/月のプランが出たので、supabaseよりも安くbranch機能が使える。
（ただしデプロイリクエストには対応していない。これが便利だったのだが。）
https://planetscale.com/blog/5-dollar-planetscale-is-here

branchはずっと稼働してると追加で5$/月かかるのだが、大体すぐ削除するのでそこまでコストかからないはず。
サポートも1回だけ使ったが割とすぐ返事くれるので今のところ印象が良い。
東京リージョンがあり、latencyも数ミリ秒で快適に使える。
代替サービスとしては検討したが採用しなかったもの

- turso
慣れているpostgresを使いたかったので
- Neon
東京リージョンがないので
- Supabase
最低25$で高いので
- CloudSQL
ちょっと高いのと管理が面倒そうなので

### CloudRun
最小インスタンスを0にすればほぼ無料で使える素晴らしいサービス。
tagをつけると、https://{tag}---hogehoge.a.run.app というURLを発行できるようになる。
tagをPR番号とかにすればPR専用のURLが作れるので、これを使用してpreview環境を構築する。

### TypeScript, Next.js, drizzle-orm, better-auth
webアプリの方はfrontend, backendともにTypeScriptを採用。
サーバーは1つで、Next.jsがbackendも担っている。
drizzle-ormでDBのmigrationとqueryを書いている。
better-authはgoogle認証で使っているが、これのoauth-proxy-pluginがpreview環境で便利なので使っている。(後述するがそのままだと使えず、魔改造している)

# 構築手順
## 事前準備
PlanetScaleのwebサイト上でDBを作り、github actionsの認証で使うservice tokenを発行する。permissionは試した感じ以下が必要。
- connect_branch
- create_branch
- delete_branch
- delete_branch_password
  branchに接続するパスワードはいちいち保存したくないので、PRのデプロイのたびにパスワードをリセットする。それに必要。
- read_branch
- write_branch_vschema
  これはなんで必要だったか忘れたが、ないとpermissionエラーになった気がする。

あとはGCPのリソースだったりDockerfileだったりはよしなに作っておく。

## github actionsのworkflowファイル作成
完成版
```yml
name: Deploy Preview Environment

on:
  pull_request:
    types: [opened, synchronize, reopened]
  workflow_dispatch:

env:
  REGION: asia-northeast1
  REPOSITORY_ID: repoid
  SERVICE_NAME: web
  PREVIEW_SERVICE_NAME: web-preview

jobs:
  deploy-preview:
    if: ${{ !contains(github.event.pull_request.labels.*.name, 'skip-preview') }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
      pull-requests: write

    steps:
      - name: Checkout
        uses: actions/checkout@8e8c483db84b4bee98b60c0593521ed34d9990e8 # v6.0.1

      - name: Setup pscale
        uses: planetscale/setup-pscale-action@d0c2789e7018488bae9d469592876b3428fa03f6 # v1 

      - name: Convert branch name
        id: branch-name
        run: |
          PSCALE_BRANCH_NAME=$(echo "${{ github.head_ref }}" | tr -cd '[:alnum:]-' | tr '[:upper:]' '[:lower:]' | cut -c1-63)
          echo "name=${PSCALE_BRANCH_NAME}" >> $GITHUB_OUTPUT

      - name: Create PlanetScale branch
        env:
          PLANETSCALE_SERVICE_TOKEN_ID: ${{ secrets.PLANETSCALE_SERVICE_TOKEN_ID }}
          PLANETSCALE_SERVICE_TOKEN: ${{ secrets.PLANETSCALE_SERVICE_TOKEN }}
        run: |
          set +e
          pscale branch show ${{ secrets.PLANETSCALE_DATABASE_NAME }} ${{ steps.branch-name.outputs.name }} --org ${{ secrets.PLANETSCALE_ORG_NAME }}
          exit_code=$?
          set -e

          if [ $exit_code -eq 0 ]; then
            echo "Branch exists. Skipping branch creation."
          else
            echo "Branch does not exist. Creating."
            pscale branch create ${{ secrets.PLANETSCALE_DATABASE_NAME }} ${{ steps.branch-name.outputs.name }} --wait --org ${{ secrets.PLANETSCALE_ORG_NAME }}
          fi

      - name: Create or reset role for branch
        id: create-role
        env:
          PLANETSCALE_SERVICE_TOKEN_ID: ${{ secrets.PLANETSCALE_SERVICE_TOKEN_ID }}
          PLANETSCALE_SERVICE_TOKEN: ${{ secrets.PLANETSCALE_SERVICE_TOKEN }}
        run: |
          ROLE_NAME="preview-pr-${{ github.event.pull_request.number }}"
          DB_NAME="${{ secrets.PLANETSCALE_DATABASE_NAME }}"
          BRANCH_NAME="${{ steps.branch-name.outputs.name }}"
          ORG_NAME="${{ secrets.PLANETSCALE_ORG_NAME }}"

          # Check if role already exists
          existing_role=$(pscale role list "$DB_NAME" "$BRANCH_NAME" --org "$ORG_NAME" -f json | jq -r --arg name "$ROLE_NAME" '.[] | select(.name == $name)')

          if [ -n "$existing_role" ]; then
            echo "Role exists. Resetting password."
            role_id=$(echo "$existing_role" | jq -r '.id')
            response=$(pscale role reset "$DB_NAME" "$BRANCH_NAME" "$role_id" --org "$ORG_NAME" --force -f json)
          else
            echo "Role does not exist. Creating."
            response=$(pscale role create "$DB_NAME" "$BRANCH_NAME" "$ROLE_NAME" --inherited-roles pg_read_all_data,pg_write_all_data,postgres --ttl 168h -f json --org "$ORG_NAME")
          fi

          host=$(echo "$response" | jq -r '.access_host_url')
          username=$(echo "$response" | jq -r '.username')
          password=$(echo "$response" | jq -r '.password')

          database_url="postgresql://$username:$password@$host/postgres?sslmode=require"
          echo "::add-mask::$password"
          echo "::add-mask::$database_url"
          echo "database_url=$database_url" >> $GITHUB_OUTPUT

      - name: Setup pnpm
        uses: pnpm/action-setup@41ff72655975bd51cab0327fa583b6e92b6d3061 # v4.2.0

      - name: Setup Node.js
        uses: actions/setup-node@395ad3262231945c25e8478fd5baf05154b1d79f # v6.1.0
        with:
          node-version: "22"
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run database migrations
        env:
          DATABASE_URL: ${{ steps.create-role.outputs.database_url }}
        run: pnpm --filter=@repo/db db:push

      - name: Run database seed
        env:
          DATABASE_URL: ${{ steps.create-role.outputs.database_url }}
        run: pnpm --filter=@repo/db db:seed

      - name: Google Auth
        id: auth
        uses: google-github-actions/auth@7c6bc770dae815cd3e89ee6cdf493a5fab2cc093 # v3.0.0
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@aa5489c8933f4cc7a4f7d45035b3b1440c9c10db # v3.0.1

      - name: Configure Docker
        run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet

      - name: Set variables
        id: vars
        run: |
          TAG="pr-${{ github.event.pull_request.number }}"
          TAG_URL=$(echo "${{ secrets.PREVIEW_URL }}" | sed "s|https://|https://${TAG}---|")
          echo "tag=${TAG}" >> $GITHUB_OUTPUT
          echo "image_tag=${{ github.sha }}" >> $GITHUB_OUTPUT
          echo "tag_url=${TAG_URL}" >> $GITHUB_OUTPUT

      - name: Create .env file
        run: |
          echo "NEXT_PUBLIC_BASE_URL=${{ steps.vars.outputs.tag_url }}" > apps/web/.env
          echo "NEXT_PUBLIC_VAPID_PUBLIC_KEY=${{ secrets.NEXT_PUBLIC_VAPID_PUBLIC_KEY }}" >> apps/web/.env

      - name: Build Docker image
        run: |
          docker build \
            --platform linux/amd64 \
            -f apps/web/Dockerfile \
            -t ${{ env.REGION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ env.REPOSITORY_ID }}/${{ env.SERVICE_NAME }}:${{ steps.vars.outputs.image_tag }} \
            .

      - name: Push Docker image
        run: |
          docker push ${{ env.REGION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ env.REPOSITORY_ID }}/${{ env.SERVICE_NAME }}:${{ steps.vars.outputs.image_tag }}

      - name: Deploy to Cloud Run preview service with tag
        id: deploy
        env:
          DATABASE_URL: ${{ steps.create-role.outputs.database_url }}
        run: |
          # プレビュー専用サービスにタグ付きでデプロイ
          gcloud run deploy ${{ env.PREVIEW_SERVICE_NAME }} \
            --project ${{ secrets.GCP_PROJECT_ID }} \
            --image ${{ env.REGION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ env.REPOSITORY_ID }}/${{ env.SERVICE_NAME }}:${{ steps.vars.outputs.image_tag }} \
            --region ${{ env.REGION }} \
            --platform managed \
            --tag ${{ steps.vars.outputs.tag }} \
            --no-traffic \
            --set-env-vars "DATABASE_URL=${DATABASE_URL},PRODUCTION_URL=${{ secrets.PRODUCTION_URL }},NEXT_PUBLIC_BASE_URL=${{ steps.vars.outputs.tag_url }},PREVIEW_URL=${{ secrets.PREVIEW_URL }},NEXT_PUBLIC_VAPID_PUBLIC_KEY=${{ secrets.NEXT_PUBLIC_VAPID_PUBLIC_KEY }}" \
            --set-secrets "BETTER_AUTH_SECRET=BETTER_AUTH_SECRET:latest,GOOGLE_CLIENT_ID=GOOGLE_CLIENT_ID:latest,GOOGLE_CLIENT_SECRET=GOOGLE_CLIENT_SECRET:latest,VAPID_PRIVATE_KEY=VAPID_PRIVATE_KEY:latest"

          echo "url=${{ steps.vars.outputs.tag_url }}" >> $GITHUB_OUTPUT

      - name: Minimize old preview comments
        uses: actions/github-script@ed597411d8f924073f98dfc5c65a23a2325f34cd # v8.0.0
        with:
          script: |
            // Get all comments on the PR
            const comments = await github.rest.issues.listComments({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
            });

            // Find old preview deployment comments (by bot and containing specific marker)
            const previewComments = comments.data.filter(comment =>
              comment.user.type === 'Bot' &&
              comment.body.includes('## Preview Environment Deployed')
            );

            // Minimize each old comment as OUTDATED using GraphQL
            for (const comment of previewComments) {
              try {
                await github.graphql(`
                  mutation($id: ID!) {
                    minimizeComment(input: {subjectId: $id, classifier: OUTDATED}) {
                      minimizedComment {
                        isMinimized
                      }
                    }
                  }
                `, {
                  id: comment.node_id
                });
                console.log(`Minimized comment ${comment.id} as OUTDATED`);
              } catch (error) {
                console.log(`Failed to minimize comment ${comment.id}: ${error.message}`);
              }
            }

      - name: Comment PR
        uses: actions/github-script@ed597411d8f924073f98dfc5c65a23a2325f34cd # 8.0.0
        with:
          script: |
            const url = '${{ steps.deploy.outputs.url }}';
            const branch = '${{ steps.branch-name.outputs.name }}';
            const body = `## Preview Environment Deployed

            | Environment | URL |
            |-------------|-----|
            | Preview | ${url} |

            **Database:** PlanetScale branch \`${branch}\` connected

            Commit: ${{ github.event.pull_request.head.sha }}`;

            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: body
            });
```

ベースは https://planetscale.com/docs/vitess/integrations/github-actions を参考にしている。

補足
- Create or reset role for branch
  DB接続用のroleを作成し、2回目の実行ですでにroleがある場合はreset passwordだけ行う。こうすることでpasswordをどこかに保存しなくて済む。
  公式ドキュメントだとMySQL用だったので、Postgresだと使用するコマンドが異なる点に注意。
  付与する権限は、pg_read_all_data,pg_write_all_data,postgresの3つ。postgresはDBのmigrationに必要で、本来ならDBのmigration用のroleとwebからの接続用のroleは分けた方がいいが、preview環境なので割愛している。

- jobを分けずに1つにしている理由
  最初はPlanetScaleのjobとCloud Runのjobを分けようと思っていたが、job間でmaskしたdatabase_urlの受け渡しができない仕様のようだったので、統一している。

## Google認証をpreview環境でも動くようにする
### preview環境でgoogle認証が動かない
preview環境のURLはPRごとに動的に変わるが、Google認証に設定するcallback urlにはwild cardは使えないので、そのままだと動かない。
こういうケースではproxyサーバーを別途用意して、callback urlをそのproxyサーバーに設定する手法がよく使われているようだった。
ただproxy用のサーバーを管理するのはちょっと面倒だなと思っていたところ、better-authにはoauth-proxyというpluginがあり、これが使えることを知った。

### oauth-proxy pluginの仕組みと問題点
本番のサーバーをproxyとして使う仕組みになっている。callback urlには本番サーバーのcallback urlを登録する。
フローとしては、
1. preview環境でログイン
2. googleにリダイレクトして認証
3. 本番サーバーにリダイレクト
4. preview環境にリダイレクト

となる。
preview環境のログインに本番サーバーを経由して大丈夫?と思ったが、本番環境ではこのproxy自体が無効化される仕組みになっている(具体的にはproductionURL=currentURLの場合は無効化)ので、あくまでpreview環境だけ特殊な動作をするという意味で問題はなさそうだった。

ただしこのplugin、本番とpreviewで同じDBを使う想定になってるらしく、DBを分けてしまうと認証ができない。具体的には3の本番サーバーにリダイレクトした時点で本番DBにuser accountが作られる。
これでは目的を達成できないので、すこしいじることにした。

### oauth-proxy pluginを改造する
やりたいことは、本番サーバーをただのproxyとして使いたいだけなので、以下のようなpluginを作った。
:::details 自作oauth-proxy
```ts
import { AuthContext, BetterAuthPlugin, InternalAdapter } from "better-auth";
import { parseSetCookieHeader } from "better-auth/cookies";
import { symmetricDecrypt, symmetricEncrypt } from "better-auth/crypto";
import { createAuthMiddleware } from "better-auth/plugins";
import { safeDestr } from "destr";

function getOrigin(url: string) {
  try {
    const parsedUrl = new URL(url);
    return parsedUrl.origin === "null" ? null : parsedUrl.origin;
  } catch {
    return null;
  }
}

type OAuthConfigSnapshot = {
  storeStateStrategy: AuthContext["oauthConfig"]["storeStateStrategy"];
  skipStateCookieCheck: AuthContext["oauthConfig"]["skipStateCookieCheck"];
  internalAdapter: InternalAdapter;
};

type AuthContextWithSnapshot = AuthContext & {
  _oauthProxySnapshot?: OAuthConfigSnapshot;
};

export interface OAuthRedirectProxyOptions {
  /**
   * The production URL where OAuth callbacks are registered.
   * This is the URL registered with OAuth providers (e.g., Google).
   */
  productionURL: string;
}

interface RedirectProxyStatePackage {
  /**
   * Original state from the OAuth provider
   */
  state: string;
  /**
   * Encrypted state cookie value for stateless verification
   */
  stateCookie: string;
  /**
   * The origin URL of the preview/staging environment
   */
  previewOrigin: string;
  /**
   * Flag to identify this as a redirect proxy request
   */
  isRedirectProxy: boolean;
}

/**
 * OAuth Redirect Proxy Plugin
 *
 * This plugin enables OAuth authentication for preview/staging environments
 * that use separate databases from production.
 *
 * Unlike the standard oauth-proxy which processes callbacks on production
 * and forwards cookies, this plugin redirects the OAuth code/state back
 * to the preview environment, allowing it to complete the authentication
 * and create users/sessions in its own database.
 *
 * Requirements:
 * - Both environments must share the same `secret`
 * - Production must have this plugin configured
 *
 * Flow:
 * 1. Preview: User initiates OAuth sign-in
 * 2. Preview: State is encrypted with preview origin info (after hook)
 * 3. OAuth Provider: Redirects to production /callback/:provider
 * 4. Production: This plugin intercepts, detects proxy state, redirects to preview
 * 5. Preview: Processes callback, creates user/session in preview DB
 */
export const oAuthRedirectProxy = (opts: OAuthRedirectProxyOptions) => {
  const productionOrigin = getOrigin(opts.productionURL);

  return {
    id: "oauth-redirect-proxy",
    hooks: {
      before: [
        {
          // On production: intercept OAuth callback and redirect to preview
          matcher(context) {
            return !!(
              context.path?.startsWith("/callback/") ||
              context.path?.startsWith("/oauth2/callback/")
            );
          },
          handler: createAuthMiddleware(async (ctx) => {
            const code = ctx.query?.code;
            const state = ctx.query?.state;
            const error = ctx.query?.error;
            const errorDescription = ctx.query?.error_description;

            if (!state || typeof state !== "string") {
              return;
            }

            // Try to decrypt state to check if it's a redirect proxy request
            let statePackage: RedirectProxyStatePackage | undefined;
            try {
              const decryptedPackage = await symmetricDecrypt({
                key: ctx.context.secret,
                data: state,
              });
              statePackage =
                safeDestr<RedirectProxyStatePackage>(decryptedPackage);
            } catch {
              // Not encrypted or not our state, continue normally
              return;
            }

            if (
              !statePackage?.isRedirectProxy ||
              !statePackage?.previewOrigin
            ) {
              return;
            }

            const previewOrigin = statePackage.previewOrigin;

            // Skip if we're already on the preview origin (avoid redirect loop)
            const requestOrigin = getOrigin(ctx.context.baseURL);
            if (requestOrigin === previewOrigin) {
              return;
            }

            // Handle OAuth errors
            if (error) {
              const errorURL =
                ctx.context.options.onAPIError?.errorURL ||
                `${previewOrigin}/error`;
              throw ctx.redirect(
                `${errorURL}?error=${encodeURIComponent(error)}${
                  errorDescription
                    ? `&error_description=${encodeURIComponent(errorDescription)}`
                    : ""
                }`,
              );
            }

            if (!code) {
              const errorURL =
                ctx.context.options.onAPIError?.errorURL ||
                `${previewOrigin}/error`;
              throw ctx.redirect(`${errorURL}?error=missing_code`);
            }

            // Extract provider from path (e.g., /callback/google -> google)
            const pathParts = ctx.context.path?.split("/") || [];
            const providerId = pathParts[pathParts.length - 1] || "google";

            // Build redirect URL to preview's callback
            const previewCallbackURL = new URL(
              `${previewOrigin}${
                ctx.context.options.basePath || "/api/auth"
              }/callback/${providerId}`,
            );

            previewCallbackURL.searchParams.set("code", code);
            // Forward the same encrypted state - preview will handle it
            previewCallbackURL.searchParams.set("state", state);

            throw ctx.redirect(previewCallbackURL.toString());
          }),
        },
        {
          // On preview: handle the redirected callback from production
          matcher(context) {
            return !!(
              context.path?.startsWith("/callback/") ||
              context.path?.startsWith("/oauth2/callback/")
            );
          },
          handler: createAuthMiddleware(async (ctx) => {
            const state = ctx.query?.state || ctx.body?.state;
            if (!state || typeof state !== "string") {
              return;
            }

            // Try to decrypt redirect proxy state package
            let statePackage: RedirectProxyStatePackage | undefined;
            try {
              const decryptedPackage = await symmetricDecrypt({
                key: ctx.context.secret,
                data: state,
              });
              statePackage =
                safeDestr<RedirectProxyStatePackage>(decryptedPackage);
            } catch {
              // Not a redirect proxy state, continue normally
              return;
            }

            if (!statePackage?.isRedirectProxy) {
              return;
            }

            // Check if we're on the preview origin
            const requestOrigin = getOrigin(ctx.context.baseURL);
            if (requestOrigin !== statePackage.previewOrigin) {
              return;
            }

            // Restore original state for oauth-proxy or normal flow
            if (statePackage.stateCookie) {
              // Decrypt the state cookie and inject it
              try {
                const stateCookieValue = await symmetricDecrypt({
                  key: ctx.context.secret,
                  data: statePackage.stateCookie,
                });
                safeDestr(stateCookieValue);

                // Snapshot original configuration for restoration in after hook
                (ctx.context as AuthContextWithSnapshot)._oauthProxySnapshot = {
                  storeStateStrategy:
                    ctx.context.oauthConfig.storeStateStrategy,
                  skipStateCookieCheck:
                    ctx.context.oauthConfig.skipStateCookieCheck,
                  internalAdapter: ctx.context.internalAdapter,
                };

                // Temporarily override findVerificationValue for database mode
                const originalAdapter = ctx.context.internalAdapter;
                ctx.context.internalAdapter = {
                  ...ctx.context.internalAdapter,
                  findVerificationValue: async (identifier: string) => {
                    if (identifier === statePackage!.state) {
                      return {
                        id: `redirect-proxy-${statePackage!.state}`,
                        identifier: statePackage!.state,
                        value: stateCookieValue,
                        createdAt: new Date(),
                        updatedAt: new Date(),
                        expiresAt: new Date(Date.now() + 10 * 60 * 1000),
                      };
                    }
                    return originalAdapter.findVerificationValue(identifier);
                  },
                };

                // Enable database mode temporarily
                ctx.context.oauthConfig.storeStateStrategy = "database";
                ctx.context.oauthConfig.skipStateCookieCheck = true;
              } catch (e) {
                ctx.context.logger.error(
                  "Failed to decrypt redirect proxy state cookie:",
                  e,
                );
                return;
              }
            }

            // Restore original state parameter
            if (ctx.query?.state) {
              ctx.query.state = statePackage.state;
            }
            if (ctx.body?.state) {
              ctx.body.state = statePackage.state;
            }
          }),
        },
      ],
      after: [
        {
          // On preview: encrypt state cookie into state parameter for cross-origin transfer
          matcher(context) {
            return !!(
              context.path?.startsWith("/sign-in/social") ||
              context.path?.startsWith("/sign-in/oauth2")
            );
          },
          handler: createAuthMiddleware(async (ctx) => {
            // Skip if we're on production
            const requestOrigin = getOrigin(ctx.context.baseURL);

            if (!requestOrigin || requestOrigin === productionOrigin) {
              return;
            }

            // Skip if header is set
            if (ctx.request?.headers.get("x-skip-oauth-proxy")) {
              return;
            }

            // Only process in stateless mode
            if (ctx.context.oauthConfig.storeStateStrategy !== "cookie") {
              return;
            }

            // Extract OAuth provider URL from sign-in response
            const signInResponse = ctx.context.returned;
            if (
              !signInResponse ||
              typeof signInResponse !== "object" ||
              !("url" in signInResponse)
            ) {
              return;
            }

            const { url: providerURL } = signInResponse;
            if (typeof providerURL !== "string") {
              return;
            }

            // Parse provider URL and extract state parameter
            const oauthURL = new URL(providerURL);
            const originalState = oauthURL.searchParams.get("state");
            if (!originalState) {
              return;
            }

            // Extract state cookie from response headers
            const headers = ctx.context.responseHeaders;
            const setCookieHeader = headers?.get("set-cookie");
            if (!setCookieHeader) {
              return;
            }

            const stateCookie = ctx.context.createAuthCookie("oauth_state");
            const parsedStateCookies = parseSetCookieHeader(setCookieHeader);
            const stateCookieAttrs = parsedStateCookies.get(stateCookie.name);
            if (!stateCookieAttrs?.value) {
              return;
            }

            const stateCookieValue = stateCookieAttrs.value;

            try {
              // Create and encrypt state package
              const statePackage: RedirectProxyStatePackage = {
                state: originalState,
                stateCookie: stateCookieValue,
                previewOrigin: requestOrigin,
                isRedirectProxy: true,
              };
              const encryptedPackage = await symmetricEncrypt({
                key: ctx.context.secret,
                data: JSON.stringify(statePackage),
              });

              // Replace state parameter with encrypted package
              oauthURL.searchParams.set("state", encryptedPackage);

              // Update response with modified URL
              ctx.context.returned = {
                ...signInResponse,
                url: oauthURL.toString(),
              };
            } catch (e) {
              ctx.context.logger.error(
                "Failed to encrypt OAuth redirect proxy state package:",
                e,
              );
              // Continue without proxy
            }
          }),
        },
        {
          // Restore OAuth config after processing callback
          matcher(context) {
            return !!(
              context.path?.startsWith("/callback") ||
              context.path?.startsWith("/oauth2/callback")
            );
          },
          handler: createAuthMiddleware(async (ctx) => {
            const contextWithSnapshot = ctx.context as AuthContextWithSnapshot;
            const snapshot = contextWithSnapshot._oauthProxySnapshot;
            if (snapshot) {
              ctx.context.oauthConfig.storeStateStrategy =
                snapshot.storeStateStrategy;
              ctx.context.oauthConfig.skipStateCookieCheck =
                snapshot.skipStateCookieCheck;
              ctx.context.internalAdapter = snapshot.internalAdapter;

              // Clear the temporary extended context value
              contextWithSnapshot._oauthProxySnapshot = undefined;
            }
          }),
        },
      ],
    },
  } satisfies BetterAuthPlugin;
};
```
:::

コメントにも書いてあるが、やっていることは、単純である。
1. preview環境でログイン。このときstateに暗号化したcallbackURLをセットする
2. googleにリダイレクトして認証 
3. 本番サーバーにリダイレクトされ、暗号化されたstateを復元して、リダイレクト先を決定
4. preview環境にリダイレクトし、通常のbetter-authの認証フローに載せる

この自作pluginや元のoauth-proxy pluginにも言えることだが、暗号化に使うsecretが本番とpreviewで同じである必要がある。これはリスクがあるので、本番運用するなら、検証環境のマスターのようなサーバーを立てておいて、それをproxyとして使うのが良いだろう。

## cleanupのworkflowを作成する
コストがかからないよう、PRがcloseされたらPlanetScaleのbranchを削除するワークフローを作成する。
また、Cloud Runのタグも削除しておくことで、無駄にURLが増えていくことを防げる。
ちなみにタグだけでなくリビジョンの削除もしようと思ったが、最新リビジョンは削除できないという仕様によりできなかった。
まあ金かかるわけではないので消す必要もないとは思う。
```yml
name: Cleanup Preview Environment

on:
  pull_request:
    types: [closed]

env:
  REGION: asia-northeast1
  PREVIEW_SERVICE_NAME: web-preview
  JOB_PREVIEW_SERVICE_NAME: job-preview

jobs:
  cleanup-planetscale:
    runs-on: ubuntu-latest
    steps:
      - name: Setup pscale
        uses: planetscale/setup-pscale-action@d0c2789e7018488bae9d469592876b3428fa03f6 # v1

      - name: Convert branch name
        id: branch-name
        run: |
          PSCALE_BRANCH_NAME=$(echo "${{ github.head_ref }}" | tr -cd '[:alnum:]-' | tr '[:upper:]' '[:lower:]' | cut -c1-63)
          echo "name=${PSCALE_BRANCH_NAME}" >> $GITHUB_OUTPUT

      - name: Delete PlanetScale branch
        env:
          PLANETSCALE_SERVICE_TOKEN_ID: ${{ secrets.PLANETSCALE_SERVICE_TOKEN_ID }}
          PLANETSCALE_SERVICE_TOKEN: ${{ secrets.PLANETSCALE_SERVICE_TOKEN }}
        run: |
          set +e
          pscale branch show ${{ secrets.PLANETSCALE_DATABASE_NAME }} ${{ steps.branch-name.outputs.name }} --org ${{ secrets.PLANETSCALE_ORG_NAME }}
          exit_code=$?
          set -e

          if [ $exit_code -eq 0 ]; then
            echo "Deleting PlanetScale branch: ${{ steps.branch-name.outputs.name }}"
            pscale branch delete ${{ secrets.PLANETSCALE_DATABASE_NAME }} ${{ steps.branch-name.outputs.name }} --org ${{ secrets.PLANETSCALE_ORG_NAME }} --force
          else
            echo "Branch does not exist. Skipping deletion."
          fi

  cleanup-preview:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - name: Google Auth
        uses: google-github-actions/auth@7c6bc770dae815cd3e89ee6cdf493a5fab2cc093 # v3.0.0
        with:
          workload_identity_provider: ${{ secrets.GCP_WORKLOAD_IDENTITY_PROVIDER }}
          service_account: ${{ secrets.GCP_SERVICE_ACCOUNT }}

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@aa5489c8933f4cc7a4f7d45035b3b1440c9c10db # v3.0.1

      - name: Remove preview tag (web)
        run: |
          TAG="pr-${{ github.event.pull_request.number }}"
          gcloud run services update-traffic ${{ env.PREVIEW_SERVICE_NAME }} \
            --project ${{ secrets.GCP_PROJECT_ID }} \
            --region ${{ env.REGION }} \
            --remove-tags ${TAG} || true

      - name: Remove preview tag (job)
        run: |
          TAG="pr-${{ github.event.pull_request.number }}"
          gcloud run services update-traffic ${{ env.JOB_PREVIEW_SERVICE_NAME }} \
            --project ${{ secrets.GCP_PROJECT_ID }} \
            --region ${{ env.REGION }} \
            --remove-tags ${TAG} || true

```