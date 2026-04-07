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

Hiện tại bot chạy ở giai đoạn PoC và xử lý như sau:

* **Gửi thông báo kênh quản lý**: gửi message vào `SLACK_RESULT_CHANNEL` nếu được cấu hình
* **Gửi DM cho manager**: thông báo trực tiếp đến manager được chọn
* **Manager list**: lấy từ `SLACK_MANAGER_LIST` trong `.env`
* **Không lưu DB**: hiện tại không lưu nội dung ứng tuyển vào database
* **Không kiểm tra duplicate**: hiện tại không có cơ chế ngăn trùng đăng ký

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
SLACK_MANAGER_LIST='[{"id":"1","full_name_english":"たまろ","slack_user_id":"XYZ...."}]' # UM list theo env
```

- `SLACK_MANAGER_LIST` là JSON array chứa danh sách UM. Mỗi phần tử cần có `id`, `full_name_english`, và `slack_user_id`.
- Khi cấu hình giá trị này, bot sẽ lấy danh sách UM từ env thay vì truy vấn database cho phần dropdown manager.

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

### Logic xử lý hiện tại

Ở giai đoạn PoC, bot không lưu dữ liệu vào database. Flow hiện tại là:

* Bot lắng nghe tin nhắn chứa `SLACK_TRIGGER_KEYWORD`
* Khi người dùng nhấn nút `応募する`, bot mở modal ứng tuyển
* Khi submit, bot gửi thông báo vào `SLACK_RESULT_CHANNEL` nếu được cấu hình
* Đồng thời bot gửi DM tới manager được chọn, lấy `slack_user_id` từ `SLACK_MANAGER_LIST`
* Không có cơ chế lưu lịch sử ứng tuyển hoặc kiểm tra trùng lặp

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

## File liên quan

* `slack.listener.ts`
* `slack.service.ts`
* `slack.module.ts`

## Tài liệu tham khảo

* [https://api.slack.com/](https://api.slack.com/)
* [https://slack.dev/bolt-js/](https://slack.dev/bolt-js/)
* [https://www.prisma.io/docs/](https://www.prisma.io/docs/)
