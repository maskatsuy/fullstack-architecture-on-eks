# fullstack-architecture-on-eks

AWS EKS上で動作するフルスタックアプリケーションのリファレンスアーキテクチャ

## プロジェクト概要

本プロジェクトは、AWS EKS（Elastic Kubernetes Service）上で動作する、フルスタックアプリケーションのリファレンス実装です。モダンなクラウドネイティブアーキテクチャのベストプラクティスを実装した学習用プロジェクトです。

## 技術スタック（予定）

### Frontend
- **TypeScript**
- **React** / **Next.js 15+** (App Router, Server Actions)
- **Apollo Client** (GraphQL)
- **Storybook** (コンポーネント開発)
- **Vitest** (テスト)
- **Turborepo** (モノレポ管理)

### BFF (Backend for Frontend)
- **Node.js** (NestJS) / **TypeScript**
- **Apollo Server** (@nestjs/graphql)
- **GraphQL** (Frontend連携)
- **gRPC** (Backend連携)

### Backend
- **Rust** (axum)
- **Node.js** (NestJS) / **TypeScript**
- **gRPC** (サービス間通信)

### Infrastructure
- **AWS** / **Amazon EKS**（Kubernetesクラスタ）
- **App Mesh** / **Istio**（サービスメッシュ）
- **CloudFront** / **Route 53** / **WAF**（CDN・DNS・セキュリティ）
- **Argo Workflows**（ワークフローエンジン）

### Event Bus
- **Amazon EventBridge**

### DevOps
- **GitHub** / **GitHub Actions**
- **Kustomize**
- **Terraform**

### Database
- **PostgreSQL**

### Authentication
- **Auth0**

### Dev Tools
- **GitHub Copilot**
- **Figma**
- **Storybook**

## アーキテクチャ

```
                 ┌─────────────┐
                 │    Auth0    │
                 │   (認証)    │
                 └─────────────┘
                        ↕
┌─────────────┐       ┌─────────────┐      ┌─────────────────┐  ┌────────────┐
│   Frontend  │──────▶│     BFF     │─────▶│   Service A     │──│ Database A │
│  (Next.js)  │GraphQL│  (NestJS)   │ gRPC │  (Rust/axum)    │  │ PostgreSQL │
│   React/TS  │       │Apollo Server│      └─────────────────┘  └────────────┘
└─────────────┘       └─────────────┘              
                             │              ┌─────────────────┐  ┌────────────┐
                             └─────────────▶│   Service B     │──│ Database B │
                                      gRPC  │ (TS/NestJS)     │  │ PostgreSQL │
                                            └─────────────────┘  └────────────┘
```

### アーキテクチャの特徴

1. **BFF中心の設計**
   - BFF (NestJS + Apollo Server) が各マイクロサービスへのアクセスを集約
   - フロントエンドは BFF とのみ通信（GraphQL）
   - Next.js Server Actions からも GraphQL 経由で BFF と通信
   - バックエンドサービス間の直接通信は最小限

2. **認証・認可**
   - **Auth0** による統一的な認証基盤
   - Frontend と BFF 間で JWT トークンを使用
   - BFF がトークン検証とバックエンドサービスへの認証情報伝播を担当

3. **技術スタック**
   - **Rust (axum)**: 高性能が必要なサービス
   - **TypeScript (NestJS)**: エンタープライズグレードのNode.js
   - **gRPC**: マイクロサービス間の効率的な通信

4. **データベース分離**
   - 各サービスが独自のデータベースを保有（マイクロサービスの原則）
   - サービス間のデータ共有はAPIを通じて実施


## セットアップ手順

### 前提条件

- AWS CLI
- kubectl
- Terraform（>= 1.12.2）
- Docker
- Node.js（>= 22）
- pnpm（>= 10.13.1）
- Rust（>=1.88.0）
- Auth0 アカウント

### Auth0 セットアップ

#### Frontend用アプリケーション
1. [Auth0](https://auth0.com/) でアカウントを作成
2. 「Create Application」をクリック
3. **Regular Web App** を選択（Next.js Server Actionsを使用するため）
4. Technology は **Next.js** を選択
5. 以下の設定を行う：
   - Application Name: `fullstack-eks-frontend`
   - Allowed Callback URLs: `http://localhost:3000/api/auth/callback`
   - Allowed Logout URLs: `http://localhost:3000`
   - Allowed Web Origins: `http://localhost:3000`
6. Domain、Client ID、Client Secret を控えておく

#### BFF/API用アプリケーション
1. 新しいアプリケーションを作成
2. **Machine to Machine Applications** を選択
3. 以下の設定を行う：
   - Application Name: `fullstack-eks-api`
   - Authorized API: 新規作成または既存のAPIを選択
4. Client ID、Client Secret を控えておく

### インストール

```bash
# リポジトリのクローン
git clone https://github.com/yourusername/fullstack-architecture-on-eks.git
cd fullstack-architecture-on-eks

# 環境変数の設定
cp .env.example .env
# .env ファイルに Auth0 の情報を記入

# 依存関係のインストール
pnpm install

# インフラストラクチャのセットアップ
cd infrastructure
terraform init
terraform plan
terraform apply

# アプリケーションのデプロイ
kubectl apply -k k8s/overlays/development
```

## 開発環境

### モノレポ構成

```
fullstack-architecture-on-eks/
├── apps/
│   ├── frontend/          # Next.js フロントエンド
│   ├── bff/              # NestJS BFF (GraphQL)
│   ├── service-a/        # Rust サービス (axum)
│   └── service-b/        # TypeScript サービス (NestJS)
├── packages/             # 共有パッケージ
│   ├── proto/           # Protocol Buffers 定義
│   └── types/           # 共有型定義
└── infrastructure/      # Terraform 設定
```

### ローカル開発

```bash
# pnpmのインストール（未インストールの場合）
npm install -g pnpm

# 依存関係のインストール（Turborepo が全てのワークスペースを管理）
pnpm install

# 全サービスの起動
pnpm dev

# 個別サービスの起動
pnpm dev --filter=frontend
pnpm dev --filter=bff
pnpm dev --filter=service-a
pnpm dev --filter=service-b

# データベースの起動（Docker Compose）
docker-compose up -d postgres

# Rust サービスのビルド
cd apps/service-a
cargo build
cargo run

# テストの実行
pnpm test

# 新しいパッケージの追加例
pnpm add axios --filter=frontend
pnpm add -D @types/node --filter=bff
```

### 環境変数

各サービスの `.env.example` を参考に設定：

```bash
# Frontend (.env.local)
NEXT_PUBLIC_GRAPHQL_URL=http://localhost:4000/graphql
AUTH0_SECRET=your-auth0-secret
AUTH0_BASE_URL=http://localhost:3000
AUTH0_ISSUER_BASE_URL=https://your-domain.auth0.com
AUTH0_CLIENT_ID=your-client-id
AUTH0_CLIENT_SECRET=your-client-secret

# BFF (.env)
AUTH0_DOMAIN=your-domain.auth0.com
AUTH0_AUDIENCE=your-api-identifier
SERVICE_A_URL=localhost:50051
SERVICE_B_URL=localhost:50052
```

## 開発ガイド

- [GitHub テンプレート使用ガイド](docs/TEMPLATES_GUIDE.md) - Issue/PRテンプレートの使い方
