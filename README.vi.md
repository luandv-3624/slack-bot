# Slack Bot - Hệ thống đăng ký tham gia dự án

Bot Slack này là một hệ thống quản lý việc đăng ký tham gia của thành viên đối với các bài tuyển dự án nội bộ.

## Tổng quan

Bot sẽ theo dõi các bài đăng tuyển dự án trong kênh Slack. Khi phát hiện bài viết chứa từ khóa chỉ định, bot sẽ tự động thêm nút "Ứng tuyển".

Khi người dùng nhấn nút, một form đăng ký chi tiết (modal) sẽ hiển thị.

## Tính năng

### 1. Theo dõi tin nhắn

* Tự động phát hiện bài viết chứa từ khóa chỉ định
* Thêm nút "Ứng tuyển" vào thread

### 2. Form đăng ký

Yêu cầu người dùng nhập các thông tin sau:

* **Loại tham gia mong muốn** (3 lựa chọn)

  * Muốn tham gia giai đoạn lên ý tưởng / đề xuất
  * Muốn tham gia với vai trò lập trình viên (implement)
  * Tạm thời chỉ quan tâm
* **Giới thiệu bản thân / động lực** (tự do nhập)
* **Manager phụ trách** (chọn từ dropdown)

### 3. Xử lý đăng ký

Thông tin đăng ký sẽ được xử lý như sau:

* **Lưu database**: ghi vào bảng ProjectApplication
* **Thông báo kênh quản lý**: gửi message vào channel chỉ định
* **Gửi DM cho manager**: thông báo trực tiếp đến manager được chọn

### 4. Ngăn chặn đăng ký trùng

Tự động ngăn việc cùng một user đăng ký nhiều lần cho cùng một project

## Cài đặt

### Thiết lập biến môi trường

Thêm vào file `.env`:

```env
# Slack Bot Configuration
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_SIGNING_SECRET=your-signing-secret
SLACK_APP_TOKEN=xapp-your-app-token

# Slack Project Applications
SLACK_RESULT_CHANNEL=C1234567890 # Channel nhận kết quả đăng ký
SLACK_TRIGGER_KEYWORD=新規プロジェクト募集 # Từ khóa trigger
```

### Cấu hình Slack App

1. **Tạo Slack App**

   * Truy cập [https://api.slack.com/apps](https://api.slack.com/apps)
   * Tạo app mới

2. **Scope cần thiết**

   * `chat:write`
   * `users:read`
   * `users:read.email`

3. **Event Subscription**

   * `message.channels`
   * `message.groups`
   * `message.mpim`
   * `message.im`

4. **Bật Socket Mode (Sử dụng cho môi trường DEV)**

   * Enable Socket Mode và lấy App Token

5. **Bot Token Scopes**

   * Cấp quyền đọc/ghi chat

### Schema Database

#### Bảng ProjectApplication

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

**Mô tả chi tiết các field:**

- `id`: ID tự động tăng, khóa chính của bảng
- `slack_user_id`: ID người dùng Slack của người ứng tuyển (ví dụ: U07KB8B8142)
- `slack_user_name`: Tên hiển thị của người dùng Slack (ví dụ: john.doe)
- `participation_type`: Loại tham gia mong muốn, có 3 giá trị:
  - `Planning`: Muốn tham gia giai đoạn lên ý tưởng / đề xuất
  - `Implementation`: Muốn tham gia với vai trò lập trình viên
  - `Interested`: Tạm thời chỉ quan tâm
- `self_introduction`: Nội dung tự giới thiệu và động lực của người ứng tuyển
- `manager_id`: ID của manager phụ trách, tham chiếu đến bảng members
- `project_message_ts`: Timestamp của tin nhắn tuyển dự án gốc trên Slack
- `project_channel`: ID kênh Slack chứa tin nhắn tuyển dự án
- `created_at`: Thời gian tạo bản ghi
- `updated_at`: Thời gian cập nhật bản ghi

**Ràng buộc duy nhất:** Kết hợp `slack_user_id` và `project_message_ts` để ngăn chặn đăng ký trùng lặp cho cùng một dự án.

#### Thêm cột vào bảng Members

```sql
ALTER TABLE members ADD COLUMN slack_user_id VARCHAR(100);
```

**slack_user_id**: ID người dùng Slack của member, dùng để gửi DM thông báo.

#### Logic lấy danh sách UM (User Manager)

Danh sách manager được lấy từ các bảng sau trong database:

- `members`: Bảng chính chứa thông tin thành viên
- `member_roles`: Bảng quan hệ giữa member và role
- `roles`: Bảng chứa định nghĩa các role

**Query logic:**
```sql
SELECT m.id, m.full_name_english
FROM members m
JOIN member_roles mr ON m.id = mr.member_id
JOIN roles r ON mr.role_id = r.id
WHERE r.code = 'um'  -- Role code cho User Manager
  AND m.is_on_leave = false  -- Loại bỏ thành viên đang nghỉ việc
ORDER BY m.full_name_english ASC;
```

Manager sẽ được hiển thị trong dropdown theo thứ tự alphabet của tên tiếng Anh.

## Cách sử dụng

### 1. Đăng bài tuyển dự án

Đăng vào Slack theo format:

```
新規プロジェクト募集

Tên dự án: XXX
Thời gian: 4/2024 - 6/2024
Số lượng: 5 người

Chi tiết...
```

![Ảnh minh họa: Bài đăng tuyển dự án với từ khóa trigger](./project-post-example.png)

*Bot sẽ tự động phát hiện từ khóa và thêm nút "Ứng tuyển" vào thread.*

### 2. Người dùng đăng ký

1. Nhấn nút "Ứng tuyển"

   ![Ảnh minh họa: Nút Ứng tuyển xuất hiện trong thread](./apply-button.png)

2. Nhập thông tin vào form

   ![Ảnh minh họa: Form đăng ký với các trường thông tin](./application-form.png)

3. Nhấn "Gửi"

   ![Ảnh minh họa: Xác nhận gửi form thành công](./form-submission-success.png)

### 3. Manager kiểm tra

* Xem danh sách đăng ký tại channel quản lý

  ![Ảnh minh họa: Thông báo trong kênh quản lý](./result-channel-notification.png)

* Nhận thông báo qua DM

  ![Ảnh minh họa: DM gửi đến manager](./manager-dm-notification.png)

## Thiết lập thông tin manager

Để nhận DM, cần set `slack_user_id` trong bảng members:

```sql
UPDATE members
SET slack_user_id = 'XYZ...'
WHERE id = ...;
```

## File liên quan

* `prisma/schema.prisma`

## Tài liệu tham khảo

* [https://api.slack.com/](https://api.slack.com/)
* [https://slack.dev/bolt-js/](https://slack.dev/bolt-js/)
* [https://www.prisma.io/docs/](https://www.prisma.io/docs/)
