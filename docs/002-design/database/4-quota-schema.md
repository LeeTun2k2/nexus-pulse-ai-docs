## Quota & Credit Domain – Database Schema

**Mục tiêu**: Quản lý quota, credit, và subscription tiers. Mỗi user có monthly credit allocation theo tier, credit được trừ khi sử dụng features (hardcode cost per feature).

Database: PostgreSQL.

---

## 1. Overview & Design Principles

- **Tier-based pricing**: free, pro, plus, unlimited (for testing)
- **Monthly credit allocation**: Mỗi tháng user nhận credits theo tier
- **Credit consumption**: Trừ credit khi sử dụng features (cost hardcode trong application)
- **Audit trail**: Tất cả credit changes được log vào `credit_transactions`
- **Future team support**: Schema hỗ trợ mở rộng cho team billing (workspace-level credits)
- **Soft delete**: Tất cả bảng có `created_at`, `updated_at`, `deleted_at`.

---

## 2. Tables

### 2.1. ENUM Types

```sql
-- Subscription tiers
CREATE TYPE subscription_tier_enum AS ENUM (
    'free',      -- Free tier
    'pro',       -- Pro tier
    'plus',      -- Plus tier
    'unlimited'  -- Unlimited tier (Optional) -> Cái này không nên, nên tạo transaction 'adjustment' để dễ quản lý. Tránh không biết tại sao bị trừ token
);

-- Credit transaction types
CREATE TYPE credit_transaction_type_enum AS ENUM (
    'initial',      -- Initial credits khi user đăng ký -> (Optional) -> Dạng free credit 300$ khi tạo account
    'monthly',      -- Monthly allocation (reset hàng tháng) -> Credit được reset hằng tháng
    'usage',        -- Credit bị trừ khi sử dụng feature  -> Credit user dùng, sẽ bị trừ
    'refund',       -- Hoàn lại credit (nếu có) -> Optional (Khi job failed, user yêu cầu hoàn tiền, admin refund)
    'bonus',        -- Bonus credits (promotion, referral) 
    'expiry',       -- Credit hết hạn (nếu có expiry policy)
    'adjustment'    -- Manual adjustment (admin) -> Admin chỉnh để test
);
```

---

### 2.2. `subscription_tiers`

**Vai trò**: Định nghĩa các subscription tiers với monthly credits và pricing.

```sql
CREATE TABLE subscription_tiers (
    id              BIGSERIAL PRIMARY KEY,
    tier            subscription_tier_enum NOT NULL UNIQUE,
    name            VARCHAR(64) NOT NULL,           -- Display name (e.g., "Free", "Pro", "Plus")
    description     TEXT,                            -- Tier description
    
    -- Monthly credit allocation
    monthly_credits INTEGER NOT NULL DEFAULT 0,     -- Credits allocated mỗi tháng
    
    -- Pricing (optional - có thể để NULL cho free tier)
    price_monthly   DECIMAL(10, 2),                  -- Monthly price
    price_annual    DECIMAL(10, 2),                  -- Annual price (nếu có discount)
    
    -- Features & limits (optional - có thể hardcode trong application)
    features        JSONB,                           -- Feature flags (e.g., {"max_resolution": "4K", "api_access": true})
    
    -- Status
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    order_index     INTEGER NOT NULL DEFAULT 0,     -- Display order (lower = higher priority)
    
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_subscription_tiers_tier ON subscription_tiers(tier) WHERE deleted_at IS NULL;
CREATE INDEX idx_subscription_tiers_active ON subscription_tiers(is_active, order_index) WHERE deleted_at IS NULL AND is_active = TRUE;
CREATE INDEX idx_subscription_tiers_deleted_at ON subscription_tiers(deleted_at);
```

**MVP**:
- **Bắt buộc**: `id`, `tier`, `name`, `monthly_credits`, `is_active`, `created_at`, `updated_at`.
- **Optional**: `description`, `price_monthly`, `price_annual`, `features`, `order_index`.

---

### 2.3. `user_billing_accounts`

**Vai trò**: Quản lý billing account của user, track current credits và subscription tier.

```sql
CREATE TABLE user_billing_accounts (
    user_id             BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    
    -- Current subscription
    subscription_tier_id BIGINT NOT NULL REFERENCES subscription_tiers(id) ON DELETE RESTRICT,
    
    -- Credit management
    current_credits     INTEGER NOT NULL DEFAULT 0,  -- Current available credits
    last_credit_reset   TIMESTAMP WITH TIME ZONE,    -- Last time credits were reset (monthly)
    
    -- Future: Team billing support
    workspace_id        BIGINT,                       -- NULL = user-level, NOT NULL = workspace-level (future)
    
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at          TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_user_billing_accounts_tier ON user_billing_accounts(subscription_tier_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_user_billing_accounts_credits ON user_billing_accounts(current_credits) WHERE deleted_at IS NULL AND current_credits > 0;
CREATE INDEX idx_user_billing_accounts_reset ON user_billing_accounts(last_credit_reset) WHERE deleted_at IS NULL;
CREATE INDEX idx_user_billing_accounts_deleted_at ON user_billing_accounts(deleted_at);
```

**MVP**:
- **Bắt buộc**: `user_id`, `subscription_tier_id`, `current_credits`, `created_at`, `updated_at`.
- **Optional**: `last_credit_reset`, `workspace_id` (future).

---

### 2.4. `credit_transactions`

**Vai trò**: Audit log immutable cho tất cả credit transactions. Track mọi thay đổi credit.

```sql
CREATE TABLE credit_transactions (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    
    -- Transaction details
    amount              INTEGER NOT NULL,             -- Credit amount (positive = added, negative = used)
    balance_after       INTEGER NOT NULL,             -- Balance after this transaction
    transaction_type    credit_transaction_type_enum NOT NULL,
    
    -- Reference to source
    reference_type      VARCHAR(32),                  -- 'request', 'job', 'subscription', 'manual', 'monthly_reset'
    reference_id        BIGINT,                       -- Reference ID (e.g., request_id, job_id, subscription_tier_id)
    
    -- Description & metadata
    description         TEXT,                         -- Transaction description
    metadata            JSONB,                        -- Additional metadata (e.g., {"feature_id": 123, "cost": 10, "feature_name": "banner"})
    
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_credit_transactions_user_id ON credit_transactions(user_id);
CREATE INDEX idx_credit_transactions_type ON credit_transactions(transaction_type);
CREATE INDEX idx_credit_transactions_reference ON credit_transactions(reference_type, reference_id) WHERE reference_type IS NOT NULL;
CREATE INDEX idx_credit_transactions_created_at ON credit_transactions(created_at DESC);
CREATE INDEX idx_credit_transactions_user_created ON credit_transactions(user_id, created_at DESC);
```

**Transaction Types** - Use Cases chi tiết:

- **`initial`**: Credits khi user đăng ký (từ tier default)
  - **Khi nào**: User đăng ký lần đầu, system assign default tier (free)
  - **Ví dụ**: User mới đăng ký → nhận 200 credits từ free tier
  - **amount**: +200 (positive)
  - **reference_type**: 'subscription'
  - **reference_id**: subscription_tier_id

- **`monthly`**: Monthly allocation (reset hàng tháng)
  - **Khi nào**: Cron job chạy hàng tháng (vd: ngày 1 mỗi tháng) để reset credits
  - **Ví dụ**: User có tier "pro" (1000 credits/tháng) → mỗi tháng reset về 1000 credits
  - **amount**: +1000 (positive, set về monthly_credits của tier)
  - **reference_type**: 'subscription'
  - **reference_id**: subscription_tier_id
  - **Lưu ý**: Không cộng dồn, mà reset về monthly_credits

- **`usage`**: Credit bị trừ khi sử dụng feature
  - **Khi nào**: User tạo request/job và feature được execute thành công
  - **Ví dụ**: User generate banner image (cost = 10 credits) → trừ 10 credits
  - **amount**: -10 (negative)
  - **reference_type**: 'request' hoặc 'job'
  - **reference_id**: request_id hoặc job_id
  - **metadata**: {"feature_id": 123, "cost": 10, "feature_name": "banner_generator"}

- **`refund`**: Hoàn lại credit
  - **Khi nào**: Job failed do lỗi hệ thống, user request refund, hoặc admin refund
  - **Ví dụ**: Job failed do lỗi AI service → refund 50 credits cho user
  - **amount**: +50 (positive)
  - **reference_type**: 'job' hoặc 'request'
  - **reference_id**: job_id hoặc request_id
  - **description**: "Refund for failed job #123"

- **`bonus`**: Bonus credits (promotion, referral)
  - **Khi nào**: User được tặng credits từ promotion, referral program, hoặc special event
  - **Ví dụ**: User giới thiệu bạn → nhận 100 bonus credits
  - **amount**: +100 (positive)
  - **reference_type**: 'promotion' hoặc 'referral'
  - **reference_id**: promotion_id hoặc referral_id
  - **description**: "Bonus credits from referral program"

- **`expiry`**: Credit hết hạn (nếu có expiry policy)
  - **Khi nào**: Credits không sử dụng hết và có expiry policy (vd: credits hết hạn sau 90 ngày)
  - **Ví dụ**: User có 50 credits không dùng trong 90 ngày → expire 50 credits
  - **amount**: -50 (negative)
  - **reference_type**: 'expiry'
  - **description**: "Credits expired after 90 days of inactivity"
  - **Lưu ý**: Có thể không cần nếu không có expiry policy

- **`adjustment`**: Manual adjustment (admin)
  - **Khi nào**: Admin điều chỉnh credits thủ công (tăng/giảm), hoặc khi user upgrade/downgrade tier
  - **Ví dụ 1**: Admin tặng 500 credits cho user do complaint → adjustment +500
  - **Ví dụ 2**: User upgrade từ free (200) lên pro (1000) → adjustment +800 (difference)
  - **amount**: Có thể positive hoặc negative
  - **reference_type**: 'manual' hoặc 'subscription'
  - **reference_id**: admin_user_id hoặc new_subscription_tier_id
  - **description**: "Manual adjustment by admin" hoặc "Tier upgrade adjustment"

**MVP**:
- **Bắt buộc**: `id`, `user_id`, `amount`, `balance_after`, `transaction_type`, `created_at`.
- **Optional**: `reference_type`, `reference_id`, `description`, `metadata`.

---

## 3. Entity Relationship Diagram (ERD)

```mermaid
erDiagram
    USERS ||--o| USER_BILLING_ACCOUNTS : "has"
    SUBSCRIPTION_TIERS ||--o{ USER_BILLING_ACCOUNTS : "subscribes"
    USERS ||--o{ CREDIT_TRANSACTIONS : "has"
    
    SUBSCRIPTION_TIERS {
        bigserial id PK "Primary Key"
        enum tier UK "free, pro, plus, unlimited"
        varchar name "Display name"
        text description "Tier description"
        integer monthly_credits "Credits allocated monthly"
        decimal price_monthly "Monthly price"
        decimal price_annual "Annual price"
        jsonb features "Feature flags"
        boolean is_active "Active status"
        integer order_index "Display order"
        timestamp created_at "Created timestamp"
        timestamp updated_at "Updated timestamp"
        timestamp deleted_at "Soft delete timestamp"
    }
    
    USER_BILLING_ACCOUNTS {
        bigint user_id PK_FK "References users(id)"
        bigint subscription_tier_id FK "References subscription_tiers(id)"
        integer current_credits "Current available credits"
        timestamp last_credit_reset "Last credit reset timestamp"
        bigint workspace_id "Workspace ID (future, nullable)"
        timestamp created_at "Created timestamp"
        timestamp updated_at "Updated timestamp"
        timestamp deleted_at "Soft delete timestamp"
    }
    
    CREDIT_TRANSACTIONS {
        bigserial id PK "Primary Key"
        bigint user_id FK "References users(id)"
        integer amount "Credit amount (positive=added, negative=used)"
        integer balance_after "Balance after transaction"
        enum transaction_type "initial, monthly, usage, refund, bonus, expiry, adjustment"
        varchar reference_type "Reference type (request, job, subscription, etc.)"
        bigint reference_id "Reference ID"
        text description "Transaction description"
        jsonb metadata "Additional metadata"
        timestamp created_at "Transaction timestamp"
    }
```

### Relationship Details:

- **USERS → USER_BILLING_ACCOUNTS**: One-to-One
  - Mỗi user có một billing account
  - Khi user bị xóa → CASCADE (billing account bị xóa)

- **SUBSCRIPTION_TIERS → USER_BILLING_ACCOUNTS**: One-to-Many
  - Một tier có thể được dùng bởi nhiều users
  - Khi tier bị xóa → RESTRICT (không cho xóa nếu còn users đang dùng)

- **USERS → CREDIT_TRANSACTIONS**: One-to-Many
  - Một user có nhiều credit transactions
  - Khi user bị xóa → CASCADE (tất cả transactions bị xóa)

---

## 4. Credit System Workflow

### 4.1. User Registration
1. User đăng ký → System assign default tier (thường là "free")
2. Tạo `user_billing_accounts` với:
   - `subscription_tier_id` = free tier
   - `current_credits` = `monthly_credits` từ tier
   - `last_credit_reset` = `NOW()`
3. Tạo `credit_transactions` với:
   - `transaction_type` = 'initial'
   - `amount` = monthly_credits
   - `balance_after` = monthly_credits

### 4.2. Monthly Credit Reset
1. Cron job chạy hàng tháng (vd: ngày 1 mỗi tháng)
2. Với mỗi user có `last_credit_reset` < current month:
   - Load `user_billing_accounts` và `subscription_tiers`
   - Calculate new credits = `monthly_credits` từ tier
   - Update `current_credits` = new credits
   - Update `last_credit_reset` = `NOW()`
   - Tạo `credit_transactions` với:
     - `transaction_type` = 'monthly'
     - `amount` = monthly_credits
     - `balance_after` = new credits
     - `reference_type` = 'subscription'
     - `reference_id` = subscription_tier_id

### 4.3. Credit Usage (Feature Consumption)
1. User tạo request → System check `current_credits` > 0
2. Calculate cost từ feature (hardcode trong application):
   - Ví dụ: banner generation = 10 credits, video generation = 50 credits
3. Nếu đủ credits:
   - Deduct credits: `current_credits` -= cost
   - Tạo `credit_transactions` với:
     - `transaction_type` = 'usage'
     - `amount` = -cost (negative)
     - `balance_after` = new balance
     - `reference_type` = 'request' hoặc 'job'
     - `reference_id` = request_id hoặc job_id
     - `metadata` = {"feature_id": ..., "cost": ..., "feature_name": ...}
4. Nếu không đủ → Block request, return error

### 4.4. Tier Upgrade/Downgrade
1. User upgrade tier → System:
   - Update `subscription_tier_id` trong `user_billing_accounts`
   - Calculate credit difference = new_monthly_credits - old_monthly_credits
   - Nếu upgrade: Add credits difference
   - Tạo `credit_transactions` với:
     - `transaction_type` = 'adjustment'
     - `amount` = credit_difference
     - `reference_type` = 'subscription'
     - `reference_id` = new_subscription_tier_id

---

## 5. Credit Cost per Feature (Hardcode)

**Lưu ý**: Cost per feature được hardcode trong application, không lưu trong DB.

**Ví dụ** (có thể config trong application):
```javascript
// Feature costs (hardcode trong application config)
const FEATURE_COSTS = {
  'banner_generator': 10,      // 10 credits per generation
  'tiktok_video': 50,          // 50 credits per generation
  'voice_over': 20,             // 20 credits per generation
  // ...
};
```

**Hoặc có thể lưu trong `generation_features.properties` JSONB**:
```json
{
  "cost_per_generation": 10,
  "max_resolution": "4K",
  "formats": ["png", "jpg"]
}
```

---

## 6. Soft Delete & Data Retention

- `subscription_tiers`:
  - Soft delete để giữ lại data cho analytics
  - Không nên xóa tier đang được dùng bởi users

- `user_billing_accounts`:
  - Soft delete khi user bị xóa (CASCADE từ users)
  - Giữ lại để audit trail

- `credit_transactions`:
  - **Immutable audit log** - không bao giờ soft delete
  - Giữ lại vĩnh viễn cho compliance và audit

---

## 7. Quan hệ với các domain khác

- **Identity Domain**:
  - `user_billing_accounts` reference `users.id`
  - `subscription_tiers` được reference từ `user_billing_accounts`

- **Generation Domain**:
  - Khi tạo request/job → System check và deduct credits
  - `credit_transactions` có thể reference `request_id` hoặc `job_id`

- **Workspace Domain**:
  - Hiện tại: Credits ở user level
  - Future: Có thể support workspace-level credits (thêm `workspace_id` vào `user_billing_accounts`)

---

## 8. MVP Considerations

### Bảng bắt buộc cho MVP:
- **`subscription_tiers`**: Định nghĩa tiers và monthly credits
- **`user_billing_accounts`**: Track current credits và tier
- **`credit_transactions`**: Audit log cho mọi credit changes

### Fields có thể bỏ cho MVP:
- `subscription_tiers.price_monthly`, `price_annual`: Có thể bỏ nếu MVP không có payment
- `subscription_tiers.features`: Có thể bỏ, hardcode trong application
- `user_billing_accounts.workspace_id`: Future, có thể bỏ cho MVP
- `credit_transactions.metadata`: Có thể bỏ nếu không cần extra info

### Monthly Reset Implementation:
- **Option 1**: Cron job chạy hàng tháng để reset credits
- **Option 2**: Lazy reset - reset khi user login và detect tháng mới
- **Option 3**: Reset khi user sử dụng và detect tháng mới

---

**Last Updated**: December 28, 2025

