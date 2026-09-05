---
title: "Rust/AxumのAPIをAWS Lambdaで動かすには？"
emoji: "🦀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Rust", "AWS", "AWSLambda", "axum"]
published: false
---

# はじめに

個人で作っている猫育成ゲームのサーバーはRust（Axum）で書いていて、AWS Lambdaで動かしています。個人開発なので、常時起動のサーバーを立てるより、リクエストが来たときだけ動くほうが安いかなと思って選びました。

ローカルでは普通のHTTPサーバーとして動かしたいので、そのへんの切り替えと、Lambda + Function URL + CloudFrontの構成、デプロイでやっていることをめもっとく。

<!-- TODO: Lambda を選ぶ前に他の選択肢（ECS/Fargate、Render、Fly.io など）を検討したか、いくらくらいで動いているかを本人が加筆 -->

# 全体の構成

```
クライアント
  ↓
CloudFront (api.example.com)
  ↓
Lambda Function URL
  ↓
Lambda (コンテナイメージ / Rust + Axum)
  ↓
Supabase PostgreSQL (pooler)
```

Lambda Function URLをそのまま使わずにCloudFrontを前に置いているのは、独自ドメインを当てたかったからです。

# ローカルとLambdaをCargo featureで切り替える

`main.rs`で`lambda`というfeatureで分岐しています。

```rust:main.rs
#[tokio::main]
async fn main() {
    rustls::crypto::aws_lc_rs::default_provider()
        .install_default()
        .expect("Failed to install default CryptoProvider");

    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();

    let config = Config::from_env();
    let app = create_app(config.clone()).await;

    #[cfg(feature = "lambda")]
    {
        lambda_http::run(app).await.expect("Lambda runtime error");
    }

    #[cfg(not(feature = "lambda"))]
    {
        let addr = format!("{}:{}", config.server_host, config.server_port);
        tracing::info!("listening on {addr}");
        let listener = tokio::net::TcpListener::bind(&addr)
            .await
            .expect("Failed to bind to address");
        axum::serve(listener, app).await.expect("Server error");
    }
}
```

`create_app`が返すAxumの`Router`をそのまま`lambda_http::run`に渡せるので、ルーティングやミドルウェアはローカルとLambdaで完全に同じものが動きます。分岐しているのはこの十数行だけです。

- `cargo build` → 普通のAxum HTTPサーバー（ローカル開発）
- `cargo build --features lambda` → Lambdaランタイム（Staging / Production）

CIでは`cargo build --features lambda`も走らせて、Lambda側のビルドが壊れていないか見ています。ローカルだけ通ってデプロイで落ちるのが一番いやなので。

# Dockerイメージを作る

Lambdaはコンテナイメージで動かしています。ビルドが遅いとつらいので`cargo-chef`で依存だけ先にキャッシュするようにしました。

```dockerfile:Dockerfile.lambda
FROM rust:1.94-bookworm AS chef
RUN cargo install cargo-chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
RUN apt-get update && apt-get install -y pkg-config libssl-dev && rm -rf /var/lib/apt/lists/*
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --features lambda --recipe-path recipe.json
COPY . .
RUN cargo build --release --features lambda --bin example-api --bin push-worker

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates libssl3 && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/example-api /usr/local/bin/bootstrap
COPY --from=builder /app/target/release/push-worker /usr/local/bin/push-worker
ENTRYPOINT ["/usr/local/bin/bootstrap"]
```

Lambdaのカスタムランタイムはバイナリの名前が`bootstrap`である必要があるので、ビルドしたバイナリを`bootstrap`という名前でコピーしています。プッシュ通知を送るワーカーも同じイメージに入れて、Lambda関数だけ分けています。

# Terraform

Lambda関数はこんな感じです。

```hcl:lambda.tf
resource "aws_lambda_function" "api" {
  function_name = "example-api"
  package_type  = "Image"
  image_uri     = "${aws_ecr_repository.api.repository_url}:latest"
  role          = aws_iam_role.lambda_execution.arn
  timeout       = 30
  memory_size   = 256

  environment {
    variables = {
      DATABASE_URL        = var.database_url
      DIRECT_DATABASE_URL = var.direct_database_url
      JWT_SECRET          = var.jwt_secret
      SSL_REQUIRE         = "true"
      RUST_LOG            = "info"
      # 以下、外部サービスのキーなどが続く
    }
  }

  lifecycle {
    ignore_changes = [image_uri]
  }
}
```

`ignore_changes = [image_uri]`が地味に大事で、イメージの更新はGitHub Actionsの`aws lambda update-function-code`でやるので、Terraformが「latestに戻す」差分を出さないようにしています。これを入れる前は、Terraformを流すたびにデプロイ済みのイメージが巻き戻る、みたいなことになりました。

Function URLは認証なしで公開して、

```hcl:lambda.tf
resource "aws_lambda_function_url" "api" {
  function_name      = aws_lambda_function.api.function_name
  authorization_type = "NONE"
}
```

CloudFrontのオリジンにします。Function URLのURLからスキーマと末尾のスラッシュを取る必要があるので、`replace`を2回かけています。

```hcl:cloudfront-api.tf
origin {
  domain_name = replace(replace(aws_lambda_function_url.api.function_url, "https://", ""), "/", "")
  origin_id   = "lambda-api"

  custom_origin_config {
    origin_protocol_policy = "https-only"
    origin_ssl_protocols   = ["TLSv1.2"]
  }
}

default_cache_behavior {
  target_origin_id       = "lambda-api"
  viewer_protocol_policy = "redirect-to-https"
  allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]

  # CachingDisabled
  cache_policy_id = "4135ea2d-6df8-44a3-9df3-4b5a84be39ad"
  # AllViewerExceptHostHeader
  origin_request_policy_id = "b689b0a8-53d0-40ab-baf2-68738e2966ac"
}
```

APIなのでキャッシュは無効（`CachingDisabled`）。origin request policyは`AllViewerExceptHostHeader`にします。Hostヘッダーをそのまま送るとFunction URL側で弾かれるので、Hostだけ除外するこのポリシーを使う必要がありました。

# デプロイ

GitHub ActionsからOIDCでAWSに入って、ECRにpushしてLambdaを更新します。

1. `aws-actions/configure-aws-credentials` でOIDC認証
2. ECRログイン
3. `Dockerfile.lambda` でビルドしてECRにpush
4. `aws lambda update-function-code` で関数を更新
5. `aws lambda wait function-updated` で更新完了を待つ
6. Lambdaを`/health`で1回呼ぶ

最後の「1回呼ぶ」は、DBマイグレーションを起動時に走らせているからです。デプロイ後に誰も叩かないと、最初のユーザーのリクエストでマイグレーションが走ることになるので、デプロイの最後に自分で叩いておきます。

Stagingは`main`にpushしたら（`server/**`が変わったときだけ）、Productionは`api/v*`のタグをpushしたら流れるようにしています。

# 起動時マイグレーションの注意点

Lambdaで起動時マイグレーションをやると、複数インスタンスが同時にコールドスタートしたときに同時にマイグレーションを始めます。なのでリトライしつつ、最終的に失敗しても`panic`せずにサーバーを起動するようにしました。

```rust:lib.rs
// Session pooler の接続上限（pool_size: 15）に達する場合があるためリトライする。
// Lambda 複数インスタンスが同時にコールドスタートすると接続が競合する。
let mut migration_success = false;
for attempt in 1..=3 {
    // ...接続してから
    match sqlx::migrate!("../../migrations").run(&direct_pool).await {
        Ok(_) => {
            tracing::info!("Migrations completed successfully (attempt {attempt})");
            migration_success = true;
            direct_pool.close().await;
            break;
        }
        Err(e) => {
            tracing::warn!("Migration attempt {attempt}/3 failed: {e}");
            direct_pool.close().await;
            if attempt < 3 {
                tokio::time::sleep(std::time::Duration::from_secs(5)).await;
            }
        }
    }
}
if !migration_success {
    // 他の Lambda インスタンスが既にマイグレーションを完了している可能性が高いため、
    // panic せずにサーバー起動を続行する。
    tracing::error!("All migration attempts failed, continuing without migration");
}
```

もうひとつ、マイグレーション用には通常のDB接続とは別の接続（`DIRECT_DATABASE_URL`）を使っています。

```rust:lib.rs
// Supavisor transaction mode (port 6543) では sqlx::migrate!() が内部で使う
// prepared statement が衝突するため、マイグレーション専用に別の接続を使う
```

Supabaseのtransaction poolerだと`sqlx::migrate!()`が内部で使うprepared statementが衝突するので、マイグレーションだけsession pooler側につなぎます。このpooler周りは別でハマった話があるので、そっちは別記事に書きました。

<!-- TODO: sqlx-supabase-pooler-random-500 を公開したら、ここに https://zenn.dev/yasi/articles/sqlx-supabase-pooler-random-500 を1行で置く -->

# おわりに

Axumの`Router`をそのまま`lambda_http::run`に渡せるので、Lambda対応で書き換えるコードはほとんどありませんでした。ローカルは普通のサーバー、本番はLambda、というのがfeature 1つで済むのは楽です。

ただ、コールドスタートの実測はちゃんと取れていないので、そこは測ってみたいなと思います。

<!-- TODO: 実際のコールドスタート時間と、体感で問題になっていないかを本人が加筆 -->

# 参考

- https://github.com/awslabs/aws-lambda-rust-runtime
- https://docs.aws.amazon.com/lambda/latest/dg/urls-configuration.html
- https://github.com/LukeMathWalker/cargo-chef
