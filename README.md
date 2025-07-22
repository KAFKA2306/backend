

# 全体設計ドキュメント ―「Lovable-for-Backend」

## 1. ビジネス目標
- **ターゲット**: バックエンドを含むフルスタック開発を素早く行いたいスタートアップ・プロダクトオーナー・小規模開発チーム。  
- **提供価値**:  
  ① チャットだけで API・データ基盤・CI/CD まで自動生成。  
  ② GitHub と Supabase をネイティブ統合し、数分で本番環境へデプロイ。  
  ③ Python のデータ処理タスクをワンクリックで追加・検証。

## 2. システムアーキテクチャ概要

```mermaid
graph TD
A[Next.js 15 Frontend] -->|SSE/REST| B(FastAPI API Gateway)
B -->|CLI 呼び出し| C[Claude Code Engine<br>Docker-in-Docker]
C -->|git push| D[GitHub Repo]
D -->|CI GitHub Actions| E[Supabase Project<br>(Postgres+Storage+Edge)]
E -->|Realtime/REST| A
C -->|Celery Task| F[Redis Queue]
F -->|結果通知| A
```

### 技術スタック
| 層 | 採用技術 | 主要役割 |
|--|--|--|
| フロントエンド | Next.js 15 (RSC), Tailwind CSS | チャットUI、進捗ログ、Notebook 埋込 |
| API ゲートウェイ | FastAPI, uvicorn, SSE | プロンプト受信、Claude Code 呼び出し |
| AI コーディング | Claude Code CLI, Anthropic API | コード生成・修正・git 操作 |
| データベース | Supabase Postgres 16 | RLS付きユーザーデータ管理 |
| ジョブキュー | Celery 5.4, Redis 7 | 長時間ビルド/ETL 実行 |
| CI/CD | GitHub Actions, Supabase CLI | マイグレーション&Edge Functions デプロイ |
| 実行環境 | Docker 26 DinD | プロジェクト隔離・安全実行 |

## 3. モジュール設計

### 3.1 Frontend (Next.js)
- **/app/chat/page.tsx**: プロンプト入力フォーム＋ストリーミング表示。  
- **/components/Notebook.tsx**: JupyterLite iframe 埋め込み。  
- **/hooks/useBuildLog.ts**: SSE で Celery 進捗を購読。  
- **i18n**: 日本語/英語自動切替。

### 3.2 API Gateway (FastAPI)
| エンドポイント | メソッド | 機能 | 認証 |
|--|--|--|--|
| /prompt | POST | Claude Code に指示送信 | Supabase JWT |
| /stream/build-log | GET (SSE) | Celery タスク進捗 | 同上 |
| /webhook/github | POST | PR イベント処理 | HMAC |
| /webhook/supabase | POST | Edge deploy 成否 | Shared secret |

### 3.3 Claude Code Engine
- **Executor**: CLI ラッパー。`--apply --commit` で自動 PR。  
- **Session Store**: Postgres `claude_sessions` テーブル。  
- **Safety Guard**: `--safe` とファイルパスホワイトリスト。

### 3.4 GitHub Actions
```
name: CI
on:
  pull_request:
    branches: ['*']
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - checkout
      - supabase db push --debug
      - supabase functions deploy --non-interactive
      - pytest
```
失敗時は GitHub Comment で Claude Code に再修正依頼。

### 3.5 Supabase Integration
- **Branching**: PR 作成時に Preview DB 自動生成。  
- **Edge Functions**: `import_reports`, `auth_webhook` 等を自動デプロイ。  
- **Storage**: バケット `assets` を作成、署名 URL で配信。

### 3.6 Data Processing (Python)
- **/jobs/etl.py**: 標準 ETL テンプレ。  
- **Celery Worker**: Docker サービス `worker`。  
- **Monitoring**: Task 成否を `etl_runs` テーブルへ記録。

## 4. データモデル（主要テーブル）

| テーブル | 主なカラム | 説明 |
|--|--|--|
| users | id, email, role | Supabase Auth 標準 |
| projects | id, owner_id, name, repo_url | GitHub リポジトリと紐付け |
| prompts | id, project_id, content, mode | plan/build/test/publish |
| ai_audit | id, project_id, cmd, ts, token_usage | 生成ログ |
| etl_runs | id, job, status, started_at, finished_at | ETL 実行履歴 |

RLS で `owner_id = auth.uid()` に制限。

## 5. フロー別処理シーケンス

### 5.1 「build」シーケンス
1. Frontend から `/prompt` に POST。  
2. API Gateway が Celery へ `build_project` タスク投入。  
3. Worker 内で Claude Code CLI 実行 → git push → PR。  
4. GitHub Actions が CI、Supabase Preview を反映。  
5. 結果を SSE 経由で UI にストリーム。

### 5.2 DB マイグレーション
1. ユーザーが「reports テーブル追加」をチャットで指示。  
2. `plan → build` で `supabase/migrations/*.sql` 生成。  
3. GitHub PR マージ時に `supabase db push` で適用。

## 6. 非機能要件対策

| 項目 | 対策 |
|--|--|
| 性能 | Redis キャッシュ＋SSE による増分配信。Claude Code 実行は並列 3 コンテナまで水平スケール。 |
| 可用性 | Supabase 99.99%、GitHub 99.9% に依存。API Gateway は Auto Healing。 |
| セキュリティ | PAT 最小権限、ネットワーク egress 制限、全操作を `ai_audit` に記録。 |
| コスト | Claude トークン量を日次集計。無料枠超過に応じた従量課金。 |

## 7. 開発ロードマップ（90日）

| フェーズ | 期間 | マイルストーン |
|--|--|--|
| POC | 0-30日 | Claude Code 呼び出し＋GitHub PR、自動 Supabase Branch 完了 |
| Alpha | 31-60日 | SSE ログ、Celery ETL、i18n、基礎 UI 完成 |
| Beta | 61-80日 | 安全サンドボックス、課金、Slack 通知統合 |
| GA | 81-90日 | SLA 監視、ドキュメント整備、公開ローンチ |

## 8. リスクと緩和策
- **LLM 生成エラー**: CLI エラーコードと stderr をキャプチャし再試行。  
- **無限ループ修正**: 同一ファイルの連続変更回数を 3 回で打ち切りアラート。  
- **コスト爆発**: トークンクォータ上限で強制停止、管理者通知。  

この全体設計により、**チャット中心の高速フルスタック開発**と**安全なバックエンド自動化**を両立できます。

