## Domain-Driven Design Overview

Tài liệu này mô tả mô hình DDD cho ứng dụng generate media phục vụ **content creator** và **marketer**.  
Mục tiêu: chia hệ thống thành các **bounded context / domain** rõ ràng, dễ mở rộng và scale lên enterprise.

Các domain chính:
- **Identity**
- **Extraction (Core GenAI)**
- **Billing**
- **Integration**
- **Workspace**

Các domain hỗ trợ / mở rộng (có thể làm phase sau):
- **Campaign / Marketing**
- **Analytics & Reporting**
- **Template Catalog & Recommendation**
- **Review & Compliance**
- **Scheduling & Notification**

Tất cả domain đều tuân theo các **cross‑cutting concerns**:
- Soft delete (`deleted_at`)
- Audit log (business events quan trọng)
- Audit fields (`created_at`, `updated_at`)

---

## 1. Identity Domain

**Mục tiêu**: Quản lý **user identity**, quyền truy cập cơ bản và chuẩn bị nền tảng để sau này mở rộng sang **team / organization**.

### 1.1. Bounded Context & Responsibility

- Đăng nhập / xác thực.
- Phase đầu có Basic login, Google Oauth2.
- Quản lý **user profile**
- Chuẩn bị để hỗ trợ:
  - **Team/Organization/Workspace-level membership**.
  - **Role/permission** ở mức user/team.

### 1.2. Aggregates & Entities

- **User**
  - Nhận diện bởi `user_id` (BIGINT).
  - Thông tin OAuth (google_id, email, name, picture, email_verified).
  - Liên kết với subscription tier hiện tại.
  - Soft delete (`deleted_at`).

- **SubscriptionTier**
  - Định nghĩa gói subscription (free, starter, pro, enterprise).
  - Chứa default credits, pricing, feature flags.
  - Dùng bởi Billing để tính quota/giới hạn.

- (Future) **Team / Organization / IdentityWorkspace**
  - Đại diện cho nhóm người dùng / tổ chức.
  - Sẽ liên kết với Workspace domain (một team có nhiều workspaces hoặc một workspace chính).

- (Future) **Membership**
  - Liên kết `User` với `Team/Workspace` + `role`.

### 1.3. Ranh giới với domain khác

- Không chứa logic Billing (chỉ reference đến `subscription_tier_id`).
- Không chứa workspace assets (chỉ cung cấp user identity và ownership).
- Không xử lý quota trực tiếp (được handle ở Billing).

---

## 2. Extraction Domain (Core GenAI)

**Mục tiêu**: Đây là **core domain** – nơi định nghĩa và thực thi các năng lực generate media / nội dung.

### 2.1. Bounded Context & Responsibility

- Định nghĩa **feature** (loại gen: banner, tiktok video, menu, voice‑over,…).
- Định nghĩa **template** (mẫu có sẵn để user chọn).
- Nhận **request** từ user (prompt, media input).
- Tạo **jobs** async để gọi các AI services.
- Sản sinh **media outputs** (image, video, audio, v.v.).

### 2.2. Aggregates & Entities

- **Category**
  - Nhóm technical capability: `text`, `image`, `video`, `voice`, ...

- **Feature**
  - Đơn vị capability (vd: `banner_generator`, `tiktok_dance_video`).
  - Có `system_prompt`, `properties` (JSONB), `media_url` demo.

- **Template**
  - Pre‑built configuration cho một feature.
  - Có metadata: trending, usage_count, tags, preview.

- **Request**
  - Aggregate gói toàn bộ **input** cho một lần generate:
    - template_prompt
    - system_prompt
    - user_prompt
    - user_media_urls
  - Gắn với Project (Workspace domain) và User (Identity).

- **Job**
  - Đơn vị execution async cho một Request.
  - Chứa `type` (image/video/voice), `status`, `input_json`.
  - Denormalized fields: `user_id`, `project_id`, `feature_id`, `category_id`.

- **Media**
  - Output của Request/Job.
  - URL tới S3 (hoặc storage khác), metadata (resolution, duration,…).
  - Phân biệt final vs intermediate bằng flags.

### 2.3. Ranh giới với domain khác

- **Không** quản lý credit/quota → Billing domain chịu trách nhiệm.
- **Không** quản lý campaign/marketing performance → Campaign/Analytics domain.
- **Không** quản lý share lên social → Integration domain.
- Chỉ tập trung vào: **từ input → AI call → output media**.

---

## 3. Billing Domain

**Mục tiêu**: Quản lý mọi thứ liên quan đến **gói cước**, **quota**, **credit**, và **thanh toán** (ở mức design, dù phase đầu có thể chỉ dùng credit).

### 3.1. Bounded Context & Responsibility

- Quản lý **credit** cho mỗi user/workspace.
- Lưu **audit log** cho mọi giao dịch credit.
- Định nghĩa **quota/giới hạn sử dụng** theo plan.
- (Future) Tích hợp với hệ thống thanh toán (Stripe, v.v.).

### 3.2. Aggregates & Entities

- **BillingAccount** (có thể map với User hoặc Workspace)
  - Đại diện cho “tài khoản thanh toán” của 1 user/workspace.
  - Có `current_credits`, `current_plan`.

- **CreditTransaction**
  - Audit log **immutable** cho mọi thay đổi credit.
  - `amount`, `balance_after`, `transaction_type`, `reference_type`, `reference_id`, `metadata`.

- **Plan / SubscriptionPlan** (hiện tại là `subscription_tiers`)
  - Quy định default credits, pricing, feature flags.

- (Future) **UsageQuota**
  - Giới hạn số jobs/requests/media per timeframe.

- (Future) **UsageCounter / UsageWindow**
  - Ghi lại actual usage để so sánh với quota.

### 3.3. Ranh giới với domain khác

- **Extraction** khi tạo Request/Job sẽ gọi vào Billing để “reserve/consume” credit.
- **Identity** chỉ tham chiếu plan của user, không xử lý logic trừ credit.
- **Workspace** có thể có BillingAccount riêng nếu sau này support team billing.

---

## 4. Integration Domain

**Mục tiêu**: Quản lý kết nối với **third‑party platforms** (social networks, storage khác, CMS, v.v.) để **share/publish** content.

### 4.1. Bounded Context & Responsibility

- Quản lý **kết nối 3rd‑party** ở mức user/workspace:
  - OAuth tokens, scopes, refresh, expiry.
- Quản lý **hành động publish/share**:
  - Post media lên Facebook/Instagram/TikTok/YouTube, v.v.
- Lưu **audit log/activities** của việc share.

### 4.2. Aggregates & Entities

- **IntegrationProvider**
  - Loại kết nối: `facebook`, `instagram`, `tiktok`, `youtube`, `x`, v.v.

- **IntegrationAccount**
  - Mapping giữa User/Workspace và một provider cụ thể.
  - Chứa tokens, status (active/revoked), metadata (page_id, channel_id,…).

- **SharedPost / PublishedContent**
  - Đại diện cho một lần publish.
  - Link tới `media` hoặc `project`.
  - Lưu trạng thái: pending, published, failed; post_id từ platform.

- (Future) **WebhookEvent**
  - Nhận callback từ platform (vd: post bị xóa, metrics update,…).

### 4.3. Ranh giới với domain khác

- Lấy **media** từ Workspace/Extraction; không tạo media.
- Gửi events sang **Analytics** để thu thập performance.
- Không chứa logic pricing/quota.

---

## 5. Workspace Domain

**Mục tiêu**: Cung cấp **không gian làm việc logic** cho content creator/marketer, nơi chứa:
- Projects
- Assets
- (tương lai) Team / Members

### 5.1. Bounded Context & Responsibility

- Tổ chức nội dung theo **workspace** (cá nhân hoặc team).
- Quản lý:
  - **Projects** – nhóm các requests/media theo mục tiêu cụ thể.
  - **Assets** – files, media, templates tuỳ biến được lưu trữ lâu dài.
  - **Folder/Collection** – nhóm assets/projects theo campaign/brand.

### 5.2. Aggregates & Entities

- (Hiện tại) **Project**
  - Container cho nhiều Requests.
  - Thuộc về User, liên kết với Feature/Category/Template.

- (Hiện tại) **Request**
  - Request cụ thể trong Project (đã mô tả ở Extraction, nhưng “sống” trong Project).

- (Hiện tại) **Media**
  - Kết quả generate từ Request/Job.
  - Được “sở hữu” bởi Project/Workspace.

- (Future) **Workspace**
  - Không gian làm việc logic (có thể đại diện cho brand/team).
  - Một user có thể belong to nhiều workspace.

- (Future) **WorkspaceMember**
  - Mapping user ↔ workspace + role.

- (Future) **Asset**
  - File/template/custom preset mà user lưu để dùng lại.

### 5.3. Ranh giới với domain khác

- Lưu trữ reference tới media (URL), nhưng không xử lý generate – phần đó là Extraction.
- Không xử lý quota – Billing giữ chuyện đó.
- Cung cấp context cho Campaign / Analytics (vd: campaign thuộc workspace X).

---

## 6. Cross‑Cutting Concerns

### 6.1. Soft Delete

- Tất cả entities domain‑critical đều có:
  - `deleted_at TIMESTAMP NULL`
- Nguyên tắc:
  - Application/query-level luôn filter `WHERE deleted_at IS NULL` cho dữ liệu “active”.
  - Hard delete chỉ áp dụng cho data tạm thời hoặc log đã archive.

### 6.2. Audit Fields

- Chuẩn hoá:
  - `created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
  - `updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP`
  - (option) `created_by`, `updated_by` (user_id) ở những bảng quan trọng.

### 6.3. Business Audit Logs

- **Billing**:
  - `credit_transactions` là audit log bắt buộc, không sửa/hard delete.

- **Integration**:
  - `shared_posts` / `published_content` + `integration_events`.

- **Workspace/Project**:
  - (Future) `workspace_activity_logs` – log các hành vi như:
    - tạo/sửa/xoá project
    - share asset
    - thay đổi member/role.

### 6.4. Denormalization & Performance

- Cho các bảng high‑volume như `jobs`, nên denormalize:
  - `user_id`, `project_id`, `feature_id`, `category_id`
  - để tối ưu truy vấn và dễ partition.

---

## 7. Supporting / Future Domains (Tóm tắt)

Những domain này có thể được thiết kế chi tiết sau, nhưng nên được mention từ sớm:

- **Campaign / Marketing Domain**
  - `campaigns`, `campaign_assets`, `campaign_metrics`.

- **Analytics & Reporting Domain**
  - `content_metrics`, `campaign_metrics`, `user_usage_analytics`.

- **Template Catalog & Recommendation Domain**
  - `global_templates`, `user_templates`, `template_collections`, `recommendation_events`.

- **Review & Compliance Domain**
  - `review_flows`, `approval_requests`, `review_comments`.

- **Scheduling & Notification Domain**
  - `content_schedules`, `notifications`, `reminders`.

---

## 8. Mapping sơ bộ sang schema hiện tại

Để tiện tracking, dưới đây là mapping nhanh giữa **schema hiện tại** và **DDD domains**:

- **Identity**
  - `users`
  - `subscription_tiers`

- **Extraction**
  - `categories`
  - `features`
  - `templates`
  - `jobs`

- **Billing**
  - `credit_transactions`
  - (dùng chung `subscription_tiers`)

- **Workspace**
  - `projects`
  - `requests`
  - `media`

- **Integration**
  - (chưa có bảng – sẽ thêm: `integration_providers`, `integration_accounts`, `shared_posts`, ... )

Các phần “(Future)” sẽ được bổ sung trong các phiên bản schema tiếp theo, nhưng tài liệu này dùng làm **kim chỉ nam** để bạn thiết kế và refactor database, service, và boundary giữa các module.


