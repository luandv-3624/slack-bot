# Slack Bot - Project Application System

このSlack Botは、社内のプロジェクト募集に対するメンバーの応募を管理するシステムです。

## 概要

Slackチャンネル内でプロジェクト募集の投稿を監視し、指定されたキーワードが含まれている場合、自動的に応募用のボタンを追加します。ユーザーがボタンをクリックすると、詳細な応募フォーム（モーダル）が表示されます。

## 機能

### 1. メッセージ監視
- 指定したキーワードを含む投稿を自動検出
- スレッドに「応募する」ボタンを追加

### 2. 応募フォーム
以下の情報の入力を求めます：
- **参加希望タイプ**（3択）
  - 企画・提案フェーズに関わりたい
  - 実装メンバーとして参加したい
  - とりあえず興味がある
- **自己紹介・意欲**（自由記入）
- **担当マネージャー**（ドロップダウン選択）

### 3. 応募処理
応募内容は以下のように処理されます：
- **データベース保存**: ProjectApplicationテーブルに記録
- **管理チャンネル通知**: 指定されたSlackチャンネルに投稿
- **マネージャーへのDM**: 選択されたマネージャーに直接通知

### 4. 重複応募防止
同一ユーザーから同じプロジェクトへの重複応募を自動で防止

## セットアップ

### 環境変数設定

`.env`ファイルに以下を追加してください：

```env
# Slack Bot Configuration
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_SIGNING_SECRET=your-signing-secret
SLACK_APP_TOKEN=xapp-your-app-token

# Slack Project Applications
SLACK_RESULT_CHANNEL=C1234567890 # 応募結果を投稿するチャンネルID
SLACK_TRIGGER_KEYWORD=新規プロジェクト募集 # 監視対象キーワード
```

### Slack App設定

1. **Slack App作成**
   - https://api.slack.com/apps にアクセス
   - 新しいアプリを作成

2. **必要なスコープ**
   - `chat:write`
   - `users:read`
   - `users:read.email`

3. **イベント購読**
   - `message.channels`
   - `message.groups`
   - `message.mpim`
   - `message.im`

4. **Socket Mode有効化 (DEV環境)**
   - Socket Mode を有効にして App Token を取得

5. **Bot Token Scopes**
   - チャットの読み取り・書き込み権限を設定

### データベーススキーマ

#### ProjectApplication テーブル
```sql
CREATE TABLE project_applications (
  id BIGINT PRIMARY KEY,
  slack_user_id VARCHAR(100) NOT NULL,
  slack_user_name VARCHAR(100) NOT NULL,
  participation_type VARCHAR(50) NOT NULL,
  self_introduction TEXT NOT NULL,
  manager_id BIGINT NOT NULL REFERENCES members(id),
  project_message_ts VARCHAR(100) NOT NULL,
  project_channel VARCHAR(100) NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(slack_user_id, project_message_ts)
);
```

**フィールド詳細説明:**

- `id`: 自動インクリメントID、テーブルの主キー
- `slack_user_id`: 応募者のSlackユーザーID（例: U07KB8B8142）
- `slack_user_name`: Slackユーザーの表示名（例: john.doe）
- `participation_type`: 参加希望タイプ、以下の3つの値:
  - `Planning`: 企画・提案フェーズに関わりたい
  - `Implementation`: 実装メンバーとして参加したい
  - `Interested`: とりあえず興味がある
- `self_introduction`: 応募者の自己紹介と意欲の内容
- `manager_id`: 担当マネージャーのID、membersテーブルを参照
- `project_message_ts`: Slack上の元のプロジェクト募集メッセージのタイムスタンプ
- `project_channel`: プロジェクト募集メッセージを含むSlackチャンネルのID
- `created_at`: レコード作成日時
- `updated_at`: レコード更新日時

**ユニーク制約:** `slack_user_id` と `project_message_ts` の組み合わせで、同じプロジェクトへの重複応募を防止。

#### Members テーブルへの追加カラム
```sql
ALTER TABLE members ADD COLUMN slack_user_id VARCHAR(100);
```

**slack_user_id**: メンバーのSlackユーザーID、DM通知の送信に使用。

#### UM（User Manager）リスト取得ロジック

マネージャーリストはデータベースの以下のテーブルから取得されます：

- `members`: メンバー基本情報テーブル
- `member_roles`: メンバーとロールの関係テーブル
- `roles`: ロール定義テーブル

**クエリロジック:**
```sql
SELECT m.id, m.full_name_english
FROM members m
JOIN member_roles mr ON m.id = mr.member_id
JOIN roles r ON mr.role_id = r.id
WHERE r.code = 'um'  -- User Managerのロールコード
  AND m.is_on_leave = false  -- 休職中のメンバーを除外
ORDER BY m.full_name_english ASC;
```

マネージャーは英語名のアルファベット順でドロップダウンに表示されます。

## 使用方法

### 1. プロジェクト募集投稿
Slackチャンネルに以下の形式で投稿：
```
新規プロジェクト募集

プロジェクト名: 〇〇システムの構築
期間: 2024年4月～6月
募集数: 5名

詳細は...
```

![画像説明: キーワードを含むプロジェクト募集投稿](./project-post-example.png)

*Botがキーワードを自動検知し、スレッドに応募ボタンを追加します。*

### 2. ユーザーが応募
1. 「応募する」ボタンをクリック

   ![画像説明: スレッドに表示された応募ボタン](./apply-button.png)

2. フォームに必要情報を入力

   ![画像説明: 応募フォームの各入力フィールド](./application-form.png)

3. 「送信」ボタンをクリック

   ![画像説明: フォーム送信完了の確認](./form-submission-success.png)

### 3. マネージャーが確認
- 管理チャンネルで全応募を確認

  ![画像説明: 管理チャンネルへの通知](./result-channel-notification.png)

- DMで個別通知を受け取り

  ![画像説明: マネージャーへのDM通知](./manager-dm-notification.png)

## マネージャー情報の設定

マネージャーがDM通知を受け取るには、Members テーブルに `slack_user_id` を設定する必要があります。

```sql
UPDATE members 
SET slack_user_id = 'XYZ...' 
WHERE id = ...;
```

## 関連ファイル

- `prisma/schema.prisma` - データベーススキーマ

## 参考リンク

- [Slack Bot developers](https://api.slack.com/)
- [Slack Bolt for JavaScript](https://slack.dev/bolt-js/)
- [Prisma Documentation](https://www.prisma.io/docs/)
