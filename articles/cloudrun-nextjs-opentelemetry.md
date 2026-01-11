---
title: "Next.js アプリを Cloud Run で動かしつつ OpenTelemetry で Cloud Trace に送信する"
emoji: "😄"
type: "tech"
topics: ["JavaScript", "TypeScript", "cloudrun"]
published: false
---

## はじめに

本番環境でアプリケーションを運用していると、「このAPIが遅い」「どこでボトルネックが発生しているのか」といった問題に直面することがあります。分散トレーシングを導入することで、リクエストがどのような経路を辿り、各処理にどれだけの時間がかかっているかを可視化できます。

この記事では、Next.js (App Router) アプリケーションを Cloud Run で動かしながら、OpenTelemetry を使って Google Cloud Trace にトレースデータを送信する方法を解説します。

### 対象読者

- Next.js を Cloud Run で運用している方
- アプリケーションのパフォーマンスを可視化したい方
- OpenTelemetry を導入したいが、どこから始めればいいかわからない方

## アーキテクチャ概要

今回構築するアーキテクチャは以下の通りです。

```
┌─────────────────────────────────────────────────────┐
│                    Cloud Run                         │
│  ┌─────────────────┐    ┌─────────────────────────┐ │
│  │   Next.js App   │───▶│  OpenTelemetry Collector │ │
│  │   (Port 3000)   │    │      (Sidecar)           │ │
│  └─────────────────┘    └───────────┬─────────────┘ │
└─────────────────────────────────────┼───────────────┘
                                      │
                                      ▼
                            ┌─────────────────┐
                            │  Cloud Trace    │
                            └─────────────────┘
```

### なぜサイドカー構成なのか

OpenTelemetry のトレースデータをバックエンドに送信する方法はいくつかありますが、Cloud Run ではサイドカーとして OpenTelemetry Collector を配置する構成がおすすめです。

**メリット:**
- アプリケーションからは `localhost` にデータを送るだけで済む
- Collector がバッチ処理やリトライを担当するため、アプリケーションの負荷が軽減される
- 認証は Collector 側で Google Cloud の認証を使うため、アプリ側での設定が不要
- 将来的にメトリクスやログも同じ Collector で扱える

## Next.js への OpenTelemetry 導入

### パッケージのインストール

```bash
pnpm add @vercel/otel @opentelemetry/instrumentation-pg
```

- `@vercel/otel`: Vercel が提供する OpenTelemetry のラッパー。Next.js との統合が簡単
- `@opentelemetry/instrumentation-pg`: PostgreSQL クエリの自動計装（使用している場合）

### instrumentation.ts の作成

Next.js の App Router では、`instrumentation.ts` を使ってサーバーサイドの初期化処理を定義できます。プロジェクトルート（`app` ディレクトリと同階層）に作成します。

```typescript
// instrumentation.ts
import { registerOTel } from "@vercel/otel";
import { PgInstrumentation } from "@opentelemetry/instrumentation-pg";

export function register() {
  console.log("[OpenTelemetry] Registering instrumentation...");
  registerOTel({
    serviceName: process.env.OTEL_SERVICE_NAME || "my-app",
    instrumentations: [new PgInstrumentation()],
  });
  console.log("[OpenTelemetry] Instrumentation registered successfully");
}
```

### 環境変数の設定

```bash
# サービス名（Cloud Trace で表示される名前）
OTEL_SERVICE_NAME=my-app

# OpenTelemetry Collector のエンドポイント（サイドカーなので localhost）
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318

# GCP プロジェクト ID（リソース属性として付与）
OTEL_RESOURCE_ATTRIBUTES=gcp.project_id=your-project-id
```

## OpenTelemetry Collector の設定

### otel-collector-config.yaml

Collector の設定ファイルを作成します。

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    send_batch_max_size: 200
    send_batch_size: 200
    timeout: 5s

  memory_limiter:
    check_interval: 1s
    limit_percentage: 65
    spike_limit_percentage: 20

  resourcedetection:
    detectors: [gcp]
    timeout: 10s

exporters:
  otlp:
    endpoint: telemetry.googleapis.com:443
    auth:
      authenticator: googleclientauth

extensions:
  health_check:
    endpoint: 0.0.0.0:13133
  googleclientauth:

service:
  extensions:
    - health_check
    - googleclientauth
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, resourcedetection, batch]
      exporters: [otlp]
```

### 設定の解説

| セクション | 説明 |
|-----------|------|
| `receivers.otlp` | アプリケーションからトレースを受け取る。HTTP (4318) と gRPC (4317) の両方に対応 |
| `processors.batch` | トレースをバッチ化して送信効率を上げる |
| `processors.memory_limiter` | メモリ使用量を制限してOOMを防ぐ |
| `processors.resourcedetection` | GCP のリソース情報（プロジェクトID、リージョン等）を自動検出 |
| `exporters.otlp` | Cloud Trace への送信。`googleclientauth` で認証 |
| `extensions.health_check` | ヘルスチェックエンドポイント（startup/liveness probe用） |
| `extensions.googleclientauth` | サービスアカウントを使った GCP 認証 |

## Cloud Run サービス定義（YAML）

Cloud Run でマルチコンテナ（サイドカー）構成を実現するには、YAML でサービスを定義し、`gcloud run services replace` でデプロイします。

### service.yaml

```yaml
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: web
  annotations:
    run.googleapis.com/ingress: all
    run.googleapis.com/invoker-iam-disabled: "true"
spec:
  template:
    metadata:
      annotations:
        # コンテナ間の依存関係を定義（web は otel-collector に依存）
        run.googleapis.com/container-dependencies: '{"web":["otel-collector"]}'
        # Secret Manager からの設定読み込み
        run.googleapis.com/secrets: 'OTEL_COLLECTOR_CONFIG:projects/YOUR_PROJECT/secrets/OTEL_COLLECTOR_CONFIG'
    spec:
      serviceAccountName: your-service-account@project.iam.gserviceaccount.com
      containers:
        # メインのアプリケーションコンテナ
        - name: web
          image: your-image:tag
          ports:
            - containerPort: 3000
          startupProbe:
            tcpSocket:
              port: 3000
            initialDelaySeconds: 5
            timeoutSeconds: 10
            periodSeconds: 10
            failureThreshold: 18
          env:
            - name: OTEL_SERVICE_NAME
              value: "my-app"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://localhost:4318"
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: "gcp.project_id=your-project-id"
            # 他の環境変数...

        # OpenTelemetry Collector サイドカー
        - name: otel-collector
          image: us-docker.pkg.dev/cloud-ops-agents-artifacts/google-cloud-opentelemetry-collector/otelcol-google:0.141.0
          args:
            - --config=/etc/otelcol-google/config.yaml
          volumeMounts:
            - name: otel-config
              mountPath: /etc/otelcol-google
              readOnly: true
          startupProbe:
            httpGet:
              path: /
              port: 13133
            initialDelaySeconds: 5
            timeoutSeconds: 10
            periodSeconds: 10
            failureThreshold: 12
          livenessProbe:
            httpGet:
              path: /
              port: 13133
            timeoutSeconds: 10
            periodSeconds: 30

      volumes:
        - name: otel-config
          secret:
            secretName: OTEL_COLLECTOR_CONFIG
            items:
              - key: latest
                path: config.yaml
```

### ポイント

1. **コンテナ依存関係**: `run.googleapis.com/container-dependencies` で、web コンテナが otel-collector に依存することを宣言。これにより、Collector が起動してから web が起動する

2. **Collector イメージ**: Google Cloud が提供する公式の Collector イメージを使用。GCP 認証がビルトイン

3. **設定の Secret 化**: Collector の設定ファイルは Secret Manager に保存し、ボリュームとしてマウント

4. **Startup Probe**: Next.js の起動には時間がかかるため、`failureThreshold` を大きめに設定

## Terraform によるインフラ設定

### 必要な API の有効化

```hcl
module "project-services" {
  source  = "terraform-google-modules/project-factory/google//modules/project_services"
  version = "~> 14.0"

  project_id = var.project_id

  activate_apis = [
    # ... 他の API ...
    "cloudtrace.googleapis.com",
    "telemetry.googleapis.com"
  ]
}
```

### IAM 権限の付与

Cloud Run のサービスアカウントに、Cloud Trace への書き込み権限を付与します。

```hcl
# Cloud Trace Agent 権限
resource "google_project_iam_member" "cloud_run_trace_agent" {
  project = var.project_id
  role    = "roles/cloudtrace.agent"
  member  = "serviceAccount:${data.google_project.project.number}-compute@developer.gserviceaccount.com"
}

# Telemetry Writer 権限
resource "google_project_iam_member" "cloud_run_telemetry_writer" {
  project = var.project_id
  role    = "roles/telemetry.tracesWriter"
  member  = "serviceAccount:${data.google_project.project.number}-compute@developer.gserviceaccount.com"
}
```

### Secret Manager への Collector 設定登録

```hcl
resource "google_secret_manager_secret" "otel_collector_config" {
  secret_id = "OTEL_COLLECTOR_CONFIG"

  replication {
    auto {}
  }
}

resource "google_secret_manager_secret_version" "otel_collector_config" {
  secret      = google_secret_manager_secret.otel_collector_config.id
  secret_data = file("${path.module}/../apps/web/cloudrun/otel-collector-config.yaml")
}
```

## GitHub Actions でのデプロイ

### デプロイワークフロー

```yaml
- name: Prepare Cloud Run service YAML
  run: |
    cp apps/web/cloudrun/service.yaml /tmp/service.yaml
    sed -i "s|WEB_IMAGE_PLACEHOLDER|${{ env.REGION }}-docker.pkg.dev/${{ secrets.GCP_PROJECT_ID }}/${{ env.REPOSITORY_ID }}/web:${{ github.sha }}|g" /tmp/service.yaml
    sed -i "s|GCP_PROJECT_ID_PLACEHOLDER|${{ secrets.GCP_PROJECT_ID }}|g" /tmp/service.yaml
    sed -i "s|SERVICE_ACCOUNT_PLACEHOLDER|${{ secrets.GCP_SERVICE_ACCOUNT_FOR_CLOUD_RUN }}|g" /tmp/service.yaml
    # 他のプレースホルダー置換...

- name: Deploy to Cloud Run
  run: |
    gcloud run services replace /tmp/service.yaml \
      --project ${{ secrets.GCP_PROJECT_ID }} \
      --region ${{ env.REGION }}
```

### なぜ `gcloud run deploy` ではなく `gcloud run services replace` か

`gcloud run deploy` コマンドではサイドカーコンテナの定義ができないため、YAML を使った `gcloud run services replace` を使用します。

## トラブルシューティング

### トレースが Cloud Trace に表示されない

1. **エンドポイントの確認**: `OTEL_EXPORTER_OTLP_ENDPOINT` が `http://localhost:4318` になっているか確認

2. **Collector のログを確認**:
   ```bash
   gcloud run services logs read web --project=YOUR_PROJECT --region=YOUR_REGION
   ```

3. **IAM 権限の確認**: サービスアカウントに `roles/cloudtrace.agent` と `roles/telemetry.tracesWriter` が付与されているか確認

4. **Collector の起動確認**: ヘルスチェック（port 13133）が正常に応答しているか確認

### Cloud Run のデプロイが失敗する

1. **Startup Probe のタイムアウト**: Next.js の起動が遅い場合、`failureThreshold` を増やす

2. **コンテナ依存関係のエラー**: `container-dependencies` の JSON 形式が正しいか確認

3. **Secret のアクセス権限**: サービスアカウントに Secret Manager へのアクセス権限があるか確認

## まとめ

この記事では、Next.js アプリケーションに OpenTelemetry を導入し、Cloud Run のサイドカー構成で Cloud Trace にトレースを送信する方法を解説しました。

### 導入によるメリット

- API エンドポイントごとのレイテンシを可視化
- DB クエリのパフォーマンスを追跡
- ボトルネックの特定が容易に

### 今後の展望

- メトリクス（`@opentelemetry/exporter-metrics-otlp-proto`）の追加
- ログとトレースの関連付け
- カスタムスパンの追加によるビジネスロジックの計装

OpenTelemetry は標準化が進んでおり、一度導入すればバックエンド（Cloud Trace、Datadog、Jaeger など）を柔軟に切り替えられます。ぜひ導入を検討してみてください。
