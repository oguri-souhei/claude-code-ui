# Claude Code UI on AWS Lightsail 設計書

## 概要

スマートフォンから Claude Code を操作するための Web UI (`@siteboon/claude-code-ui`) を AWS Lightsail でホスティングする。

## アーキテクチャ

```
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

| コンポーネント | 技術                            |
| -------------- | ------------------------------- |
| コンテナ基盤   | AWS Lightsail Container Service |
| ベースイメージ | node:20-bookworm-slim           |
| プロセス管理   | PM2                             |
| Web UI         | @siteboon/claude-code-ui        |
| CLI            | @anthropic-ai/claude-code       |

## インフラ構成

### Lightsail Container Service

| 項目       | 値                          |
| ---------- | --------------------------- |
| サービス名 | claude-code-ui              |
| Power      | Nano (512MB RAM, 0.25 vCPU) |
| Scale      | 1                           |
| リージョン | ap-northeast-1 (東京)       |
| 月額概算   | $7                          |

### ネットワーク

| 項目       | 値                                                  |
| ---------- | --------------------------------------------------- |
| 公開ポート | 3001                                                |
| プロトコル | HTTPS (Lightsail 自動対応)                          |
| ドメイン   | \*.ap-northeast-1.cs.amazonlightsail.com (自動発行) |

## ファイル構成

```
claude-code-ui-docker/
├── Dockerfile
├── docker-compose.yml      # ローカル開発用
├── .env.example
├── .dockerignore
└── deployment/
    └── container.json      # Lightsail デプロイ設定
```

## Dockerfile

```dockerfile
FROM node:20-bookworm-slim

# メタデータ
LABEL maintainer="souhei"
LABEL description="Claude Code UI for mobile access"

# ビルド依存関係（node-pty のコンパイルに必要）
RUN apt-get update && apt-get install -y \
    python3 \
    make \
    g++ \
    git \
    && rm -rf /var/lib/apt/lists/*

# グローバルパッケージをインストール
RUN npm install -g \
    pm2 \
    @anthropic-ai/claude-code \
    @siteboon/claude-code-ui

# Claude 設定ディレクトリを作成
RUN mkdir -p /root/.claude

# 作業ディレクトリ
WORKDIR /workspace

# ポート公開
EXPOSE 3001

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3001/ || exit 1

# PM2 でプロセス管理
CMD ["pm2-runtime", "claude-code-ui"]
```

## docker-compose.yml（ローカル開発用）

```yaml
version: "3.8"

services:
  claude-code-ui:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: claude-code-ui
    ports:
      - "3001:3001"
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
    volumes:
      - ./workspace:/workspace
      - claude-config:/root/.claude
    tty: true
    stdin_open: true
    restart: unless-stopped

volumes:
  claude-config:
```

## .env.example

```bash
# Anthropic API Key
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
```

## .dockerignore

```
.git
.gitignore
*.md
.env
node_modules
```

## deployment/container.json（Lightsail デプロイ設定）

```json
{
  "serviceName": "claude-code-ui",
  "containers": {
    "claude-code-ui": {
      "image": ":claude-code-ui.latest",
      "environment": {
        "ANTHROPIC_API_KEY": "<API_KEY>"
      },
      "ports": {
        "3001": "HTTP"
      }
    }
  },
  "publicEndpoint": {
    "containerName": "claude-code-ui",
    "containerPort": 3001,
    "healthCheck": {
      "healthyThreshold": 2,
      "unhealthyThreshold": 2,
      "timeoutSeconds": 5,
      "intervalSeconds": 30,
      "path": "/",
      "successCodes": "200-499"
    }
  }
}
```

## デプロイ手順

### 前提条件

- AWS CLI インストール済み
- AWS 認証情報設定済み (`aws configure`)
- Docker インストール済み

### 1. プロジェクトディレクトリ作成

```bash
mkdir -p ~/dev/claude-code-ui-docker/deployment
cd ~/dev/claude-code-ui-docker
```

### 2. ファイル作成

上記の Dockerfile, docker-compose.yml, .env.example, .dockerignore, deployment/container.json を作成

### 3. ローカルでビルド＆テスト

```bash
# .env ファイル作成
cp .env.example .env
# ANTHROPIC_API_KEY を設定

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

### 4. Lightsail Container Service 作成

```bash
# コンテナサービス作成
aws lightsail create-container-service \
  --service-name claude-code-ui \
  --power nano \
  --scale 1 \
  --region ap-northeast-1

# 作成完了まで待機（数分かかる）
aws lightsail get-container-services \
  --service-name claude-code-ui \
  --region ap-northeast-1
```

### 5. イメージをプッシュ

```bash
# Lightsail にイメージをプッシュ
aws lightsail push-container-image \
  --service-name claude-code-ui \
  --label latest \
  --image claude-code-ui:latest \
  --region ap-northeast-1
```

### 6. デプロイ

```bash
# container.json の <API_KEY> を実際の値に置換してからデプロイ
aws lightsail create-container-service-deployment \
  --service-name claude-code-ui \
  --cli-input-json file://deployment/container.json \
  --region ap-northeast-1
```

### 7. URL 確認

```bash
aws lightsail get-container-services \
  --service-name claude-code-ui \
  --region ap-northeast-1 \
  --query 'containerServices[0].url' \
  --output text
```

## セキュリティ考慮事項

### 実装済み

| 項目         | 対策                          |
| ------------ | ----------------------------- |
| 通信暗号化   | Lightsail が HTTPS を自動提供 |
| API キー管理 | 環境変数で注入                |

### 追加推奨

| 項目             | 対策                                  | 優先度 |
| ---------------- | ------------------------------------- | ------ |
| アクセス制限     | claude-code-ui の認証機能を利用       | 高     |
| IP 制限          | Lightsail Firewall で特定 IP のみ許可 | 中     |
| カスタムドメイン | 独自ドメイン + ACM 証明書             | 低     |

### IP 制限の設定（オプション）

```bash
# 自宅 IP のみ許可（例）
aws lightsail update-container-service \
  --service-name claude-code-ui \
  --public-domain-names '{}' \
  --region ap-northeast-1
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
# イメージ更新時
docker compose build
aws lightsail push-container-image \
  --service-name claude-code-ui \
  --label latest \
  --image claude-code-ui:latest \
  --region ap-northeast-1

aws lightsail create-container-service-deployment \
  --service-name claude-code-ui \
  --cli-input-json file://deployment/container.json \
  --region ap-northeast-1
```

### サービス削除

```bash
aws lightsail delete-container-service \
  --service-name claude-code-ui \
  --region ap-northeast-1
```

## コスト見積もり

| 項目                       | 月額         |
| -------------------------- | ------------ |
| Lightsail Container (Nano) | $7           |
| データ転送（1GB まで無料） | $0           |
| **合計**                   | **約 $7/月** |

## 次のステップ

1. [ ] ローカルで Docker ビルド＆テスト
2. [ ] AWS CLI セットアップ確認
3. [ ] Lightsail Container Service 作成
4. [ ] イメージプッシュ＆デプロイ
5. [ ] 動作確認
6. [ ] （オプション）IP 制限設定

## 参考リンク

- [AWS Lightsail Container Services](https://docs.aws.amazon.com/lightsail/latest/userguide/amazon-lightsail-container-services.html)
- [@siteboon/claude-code-ui](https://www.npmjs.com/package/@siteboon/claude-code-ui)
- [@anthropic-ai/claude-code](https://www.npmjs.com/package/@anthropic-ai/claude-code)
