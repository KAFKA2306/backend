
# Lovable Backend Lite  
**完全具体ガイド＆完全具体証書**  
Version 1.0 – 2025-07-22

## 目次
1. ユーザー向け完全具体ガイド  
2. 開発者向け完全具体証書  
&nbsp; &nbsp;2.1 システム全体構成  
&nbsp; &nbsp;2.2 リポジトリ構造  
&nbsp; &nbsp;2.3 サーバーレス関数インターフェース  
&nbsp; &nbsp;2.4 CI/CD パイプライン  
&nbsp; &nbsp;2.5 環境変数定義  
&nbsp; &nbsp;2.6 運用オペレーション  
&nbsp; &nbsp;2.7 トラブルシューティング  

## 1. ユーザー向け完全具体ガイド

### 1.1 事前準備
| 要素 | 内容 |
| :-- | :-- |
| ブラウザ | Chrome/Edge 最新版 |
| GitHub アカウント | Public リポジトリ作成権限があること |
| AI API キー | Anthropic Claude または OpenAI (Bring-Your-Own-Key) |
| DB 抱き合わせ不要 | Neon DB は自動プロビジョニングされる |

### 1.2 サインアップ手順
1. Lovable Backend Lite トップページにアクセス。  
2. **Sign in with GitHub** をクリックし、GitHub OAuth を承認。  
3. プロンプトで **GitHub App** のインストール先を「Only select repositories」に設定し、空のリポジトリを指定または新規作成。  
4. ダッシュボードが開いたら、右上のプロフィールメニューで **AI Provider → API Key** を入力し保存。

### 1.3 新規バックエンド作成
1. **Create Backend** ボタンを押下。  
2. 表示されたモーダルで  
   - Repository Name: `awesome-backend`  
   - Visibility: `Public`  
   を入力し **Create** を押す。  
3. 10秒以内にリポジトリ作成と初期コミットが完了し、画面にリポジトリ URL が表示される。

### 1.4 API ＆ DB 生成
1. チャット欄に次のように入力  
   ```
   ユーザー認証 API と users テーブルを作成して
   ```
2. 右下の **Send** をクリック。  
3. 画面左側の **Jobs** パネルでステータスが `QUEUED → RUNNING → COMPLETED` と変化するのを確認。  
4. GitHub に `feature/auth` ブランチの PR が自動生成され、コメント欄に **Preview URL** が貼られる。

### 1.5 プレビュー確認
1. PR の **Preview URL** をクリック。  
2. `/api/auth/signup` で POST 送信しステータス200が返ることを確認。  
3. `/api/auth/login` でトークンが返却されることを確認。

### 1.6 マージ＆本番利用
1. PR の **Checks** が success になっていることを確認し **Merge**。  
2. main ブランチが自動デプロイされ、PR コメント欄に **Production URL** が追記される。  
3. 本番 URL で API を利用開始。

### 1.7 追加開発コマンド例
| 目的 | チャットコマンド | 効果／備考 |
| :-- | :-- | :-- |
| 商品一覧 API | `products API を GET/POST で作成` | `/products` 生成 |
| カラム追加 | `users に last_login:datetime を追加` | Prisma migration 生成 |
| テスト生成 | `上記 API の pytest も書いて` | `tests/` に test_xxx.py |

### 1.8 よくある質問
- **API キーが漏えいした場合？** → Settings → API Keys → **Revoke** をクリックし、再発行してください。  
- **Preview の起動が遅い** → Railway free インスタンスはコールドスタートが発生します。30秒以上遅い場合は再実行してください。  
- **Private リポジトリで使える？** → GitHub Actions の無料枠が課金対象になるため、ご利用前に設定をご確認ください。

## 2. 開発者向け完全具体証書

### 2.1 システム全体構成
| レイヤー | サービス | 主な役割 |
| :-- | :-- | :-- |
| フロントエンド | Vercel Static Export (Next.js 14) | UI, OAuth Callback, SSE |
| エッジ関数 | Cloudflare Workers | `/generate_api`, `/github_webhook` |
| AI モデル | BYOK Claude v3 / GPT-4o | コード生成 |
| ソース管理 | GitHub (App) | PR, Actions, Checks, Secrets |
| プレビュー | Railway Free Plan | PR ごとの FastAPI コンテナ |
| 本番 API | Railway | main ブランチ FastAPI |
| DB | Neon Serverless Postgres | 自動スキーマ適用 |

### 2.2 リポジトリ構造
```
repo-root/
├── app/               # FastAPI ソース
│   ├── api/           # 生成エンドポイント
│   └── models/        # Prisma スキーマ由来
├── prisma/
│   ├── schema.prisma  # DB 定義
│   └── migrations/
├── tests/             # pytest
├── .github/
│   ├── workflows/
│   │   └── deploy.yml
│   └── CLAUDE_PROMPT.md
└── README.md
```

### 2.3 サーバーレス関数インターフェース

#### 2.3.1 `POST /generate_api`
| フィールド | 型 | 必須 | 説明 |
| :-- | :-- | :-- | :-- |
| `prompt` | string | ✔ | 自然言語指示 |
| `repo` | string | ✔ | `org/name` |
| `branch` | string | ✖ | 省略時は `feature/$hash` |
| `db_url` | string | ✖ | Neon connection string |

- 202 Accepted を返し、`job_id` を JSON で返却。  
- Cloudflare **KV** にジョブを格納、SSE チャンネル `/events/{job_id}` で進捗を push。

#### 2.3.2 `POST /github_webhook`
受信イベント  
- `pull_request` → Preview URL を取得し PR へコメント  
- `workflow_run` → Checks 状態を UI に転送

### 2.4 CI/CD パイプライン (`.github/workflows/deploy.yml`)
1. `on: [pull_request]`  
2. **Setup**: `docker build . -t $IMAGE`  
3. **Push**: Railway CLI で PR 番号ごとに service create/update  
4. **Post**: Preview URL を GitHub API で PR コメント  
5. `on: [push]` (main) では同じ手順で Production service update

### 2.5 環境変数定義
| 変数 | 配置 | 用途 |
| :-- | :-- | :-- |
| `AI_PROVIDER` | Workers | `claude` or `openai` |
| `AI_API_KEY` | Workers (Secret) | 生成 API 認証 |
| `GITHUB_APP_ID` | Workers Secret | App ID |
| `GITHUB_APP_PK` | Workers Secret | PEM |
| `NEON_DATABASE_URL` | Railway | Postgres 接続 |
| `SECRET_KEY` | FastAPI | JWT 署名 |

### 2.6 運用オペレーション

#### 2.6.1 デプロイ確認
```
gh run watch -R org/repo
```
成功後、Railway で `lovable-prod` コンテナが `Running` であることを確認。

#### 2.6.2 ロールバック
```
git revert -m 1 <merge_commit>
git push origin main
```
Actions が過去バージョンを再デプロイ。

#### 2.6.3 モニタリング
- **Workers**: Cloudflare Dashboard → Analytics → Workers  
- **Railway**: Service → Metrics (CPU, Mem)  
- **DB**: Neon Console → Insights

### 2.7 トラブルシューティング
| 症状 | 原因候補 | 対処 |
| :-- | :-- | :-- |
| Preview URL が 404 | Railway build 失敗 | PR の Checks log を確認し再デプロイ |
| Cold Start >30s | Railway sleep | Paid plan upgrade or keep-alive ping |
| AI 生成が空返却 | API キー失効 | Dashboard → Re-enter Key |
| DB Migration error | Prisma Conflict | `prisma migrate resolve --applied <name>` で状態修正 |



