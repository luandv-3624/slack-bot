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
現在のPoCでは、応募は以下のように処理されます：
- **管理チャンネル通知**: `SLACK_RESULT_CHANNEL`が設定されていれば通知を送信
- **マネージャーへのDM**: 選択されたマネージャーに直接通知
- **マネージャーリスト**: `SLACK_MANAGER_LIST`から取得
- **データ保存なし**: 現在は応募内容をデータベースに保存しません
- **重複チェックなし**: 現在は重複応募の制御を行いません

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
SLACK_MANAGER_LIST='[{"id":"1","full_name_english":"たまろ","slack_user_id":"XYZ..."}]' # envからUMのリストを指定
```

- `SLACK_MANAGER_LIST` は `id`, `full_name_english`, `slack_user_id` を持つ JSON 配列です。
- これを設定すると、ボットは manager dropdown を DB からではなく env から取得します。

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

4. **Socket Mode有効化**
   - Socket Mode を有効にして App Token を取得

5. **Bot Token Scopes**
   - チャットの読み取り・書き込み権限を設定

### ロジック

現在のPoCでは、Botの動作は以下のとおりです：

- `SLACK_TRIGGER_KEYWORD`を含む投稿を検知
- 「応募する」ボタンを押すとモーダルが開く
- 送信時に`SLACK_RESULT_CHANNEL`へ通知を送信（設定されていれば）
- 選択されたマネージャーへ、`SLACK_MANAGER_LIST`に設定されたSlackユーザーIDでDMを送信
- 応募内容はデータベースに保存しない
- 重複応募のチェックは行わない

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


## 関連ファイル

- `slack.listener.ts` - メインロジック
- `slack.service.ts` - Slack Boltアプリ初期化
- `slack.module.ts` - NestJSモジュール定義

## 参考リンク

- [Slack Bot developers](https://api.slack.com/)
- [Slack Bolt for JavaScript](https://slack.dev/bolt-js/)
- [Prisma Documentation](https://www.prisma.io/docs/)
