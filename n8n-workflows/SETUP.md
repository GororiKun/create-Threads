# n8n セットアップガイド

## 必要な準備

### 1. GitHub Personal Access Token を発行
1. GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. **Generate new token** をクリック
3. スコープ: `repo` にチェック（フルアクセス）
4. トークンをコピーして保管

### 2. Threads（Meta）アクセストークン確認
- Meta for Developers で発行済みのアクセストークンを手元に用意
- 必要な権限: `threads_basic`, `threads_content_publish`, `threads_read_engagement`
- 有効期限を確認（60日トークンの場合は定期更新が必要）

---

## n8n Cloud セットアップ手順

### Step 1: 認証情報（Credentials）を登録

n8n Cloud にログイン → **Credentials** → **Add Credential** から以下を2つ追加：

#### ① GitHub Token（HTTP Header Auth）
| 項目 | 値 |
|------|-----|
| Credential Name | `GitHub Token` |
| Type | `Header Auth` |
| Name | `Authorization` |
| Value | `Bearer {あなたのGitHub Personal Access Token}` |

#### ② Threads Access Token（HTTP Header Auth）
| 項目 | 値 |
|------|-----|
| Credential Name | `Threads Access Token` |
| Type | `Header Auth` |
| Name | `Authorization` |
| Value | `Bearer {あなたのThreadsアクセストークン}` |

---

### Step 2: ワークフローをインポート

1. n8n Cloud → **Workflows** → **Add Workflow** → **Import from file**
2. `01_posting_workflow.json` をインポート → 保存 → **Activate**
3. `02_data_collection_workflow.json` をインポート → 保存 → **Activate**

---

### Step 3: 認証情報を各ノードに紐付け

インポート後、各HTTPリクエストノードを開いて **Credential** を選択する：
- `承認済み投稿を取得` → `GitHub Token`
- `Threadsコンテナ作成` → `Threads Access Token`
- `Threads投稿公開` → `Threads Access Token`
- `post_log取得` / `GitHubにpost_logを保存` → `GitHub Token`
- `Threadsインサイト取得` → `Threads Access Token`
- `daily_metrics取得` / `GitHubにdaily_metricsを保存` → `GitHub Token`

---

## ワークフロー動作スケジュール

| ワークフロー | スケジュール | UTC | 内容 |
|---|---|---|---|
| 投稿実行 | 毎日 7:00 JST | 22:00 | 朝の投稿を実行 |
| 投稿実行 | 毎日 12:00 JST | 03:00 | 昼の投稿を実行 |
| 投稿実行 | 毎日 21:00 JST | 12:00 | 夜の投稿を実行 |
| データ収集 | 毎日 12:00 JST | 03:00 | 過去7日の投稿メトリクスを収集 |

---

## フロー全体図

```
【月曜 7:00 JST】
Claude Code（秘書オーケストレーター）
  ↓ 計画・リサーチ・執筆・承認
  ↓ approved_posts.json を GitHub に書き込む

【毎日 7:00/12:00/21:00 JST】
n8n ワークフロー①（投稿実行）
  ↓ GitHub から approved_posts.json を読み込む
  ↓ Threads API で投稿
  ↓ GitHub の post_log.json に結果を記録

【毎日 12:00 JST】
n8n ワークフロー②（データ収集）
  ↓ GitHub から post_log.json を読み込む
  ↓ Threads Insights API でメトリクス取得
  ↓ GitHub の daily_metrics.json に保存

【毎週日曜 → 月曜 7:00 JST】
Claude Code（分析担当 → 秘書）
  ↓ daily_metrics.json を読み込んで週次分析
  ↓ analysis_report.json に改善方針を書き込む
  ↓ 次週サイクルへ
```

---

## Threadsアクセストークンの更新

Meta のアクセストークンは60日で失効します。更新は以下で自動化できます：

n8n に3つ目のワークフロー（月1回実行）を追加：
```
GET https://graph.threads.net/refresh_access_token
  ?grant_type=th_refresh_token
  &access_token={現在のトークン}
→ 新しいトークンを取得 → n8n Credentials を更新
```

または Meta Developer Console で Long-lived Token（60日）を手動更新。

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| 投稿されない | approved_posts.jsonにstatusが'approved'でない | Claudeの秘書エージェントの承認ステータスを確認 |
| GitHub 401エラー | トークン期限切れ or スコープ不足 | Personal Access Token を再発行 |
| Threads 400エラー | テキストが空 or 文字数超過（最大500文字） | 原稿の文字数を確認 |
| Threads 401エラー | アクセストークン失効 | Meta Developer Console でトークン更新 |
| メトリクスが0 | 投稿後すぐは集計されない（数時間待ちが必要） | 正常。24時間後に再収集される |
