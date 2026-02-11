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
    curl \
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
