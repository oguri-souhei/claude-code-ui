# Claude Code UI on AWS Lightsail

スマートフォンから Claude Code を操作するための Web UI を AWS Lightsail でホスティングする。

## アーキテクチャ

```text
┌─────────────┐     HTTPS      ┌──────────────────────────────────┐
│   iPhone    │ ◄────────────► │  AWS Lightsail Container Service │
│  (Browser)  │                │  ┌────────────────────────────┐  │
└─────────────┘                │  │     Docker Container       │  │
                               │  │  ┌──────────────────────┐  │  │
                               │  │  │   claude-code-ui     │  │  │
                               │  │  │   (PM2 runtime)      │  │  │
                               │  │  ├──────────────────────┤  │  │
                               │  │  │   Claude CLI         │  │  │
                               │  │  │   (node-pty)         │  │  │
                               │  │  └──────────────────────┘  │  │
                               │  └────────────────────────────┘  │
                               └──────────────────────────────────┘
```

## 技術スタック

| コンポーネント | 技術 |
|---------------|------|
| コンテナ基盤 | AWS Lightsail Container Service |
| ベースイメージ | node:20-bookworm-slim |
| プロセス管理 | PM2 |
| Web UI | @siteboon/claude-code-ui |
| CLI | @anthropic-ai/claude-code |

## セットアップ

### 前提条件

- Docker
- AWS CLI（デプロイ時）
- Anthropic API Key

### ローカル開発

```bash
cd docker

# 環境変数を設定
cp .env.example .env
# .env を編集して ANTHROPIC_API_KEY を設定

# ビルド
docker compose build

# 起動
docker compose up -d

# 動作確認
curl http://localhost:3001

# ログ確認
docker compose logs -f

# 停止
docker compose down
```

## AWS Lightsail へのデプロイ

### 1. Container Service 作成

```bash
aws lightsail create-container-service \
  --service-name claude-code-ui \
  --power nano \
  --scale 1 \
  --region ap-northeast-1
```

### 2. イメージをプッシュ

```bash
aws lightsail push-container-image \
  --service-name claude-code-ui \
  --label latest \
  --image claude-code-ui:latest \
  --region ap-northeast-1
```

### 3. デプロイ

```bash
# deployment/container.json の <API_KEY> を実際の値に置換してから実行
aws lightsail create-container-service-deployment \
  --service-name claude-code-ui \
  --cli-input-json file://docker/deployment/container.json \
  --region ap-northeast-1
```

### 4. URL 確認

```bash
aws lightsail get-container-services \
  --service-name claude-code-ui \
  --region ap-northeast-1 \
  --query 'containerServices[0].url' \
  --output text
```

## 運用

### ログ確認

```bash
aws lightsail get-container-log \
  --service-name claude-code-ui \
  --container-name claude-code-ui \
  --region ap-northeast-1
```

### 再デプロイ

```bash
cd docker
docker compose build

aws lightsail push-container-image \
  --service-name claude-code-ui \
  --label latest \
  --image claude-code-ui:latest \
  --region ap-northeast-1

aws lightsail create-container-service-deployment \
  --service-name claude-code-ui \
  --cli-input-json file://docker/deployment/container.json \
  --region ap-northeast-1
```

### サービス削除

```bash
aws lightsail delete-container-service \
  --service-name claude-code-ui \
  --region ap-northeast-1
```

## コスト

| 項目 | 月額 |
|-----|------|
| Lightsail Container (Nano) | $7 |
| データ転送（1GB まで無料） | $0 |
| **合計** | **約 $7/月** |

## ディレクトリ構成

```text
.
├── README.md
├── doc/
│   └── requirement.md
└── docker/
    ├── Dockerfile
    ├── docker-compose.yml
    ├── .env.example
    ├── .dockerignore
    ├── .gitignore
    └── deployment/
        └── container.json
```
