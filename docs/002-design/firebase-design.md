# Firebase Design – Sidecar Sync Pattern

**Mục tiêu**: Firebase Realtime Database đóng vai trò là lớp "Signaling" (báo hiệu trạng thái nóng) cho real-time job status updates. PostgreSQL vẫn là "Source of Truth" cho persistent data.

---

## 1. Architectural Pattern: "The Sidecar Sync"

### Data Flow

- **Write Path**: Client → API (Node/Go) → Postgres (Create Job) → Push Job ID sang Firebase
- **Read Path (List/History)**: Client → API → Postgres
- **Read Path (Real-time Status)**: Client → Subscribe Firebase path `/user_jobs/{user_id}/{job_id}`

### Benefits

1. **Code Frontend cực nhàn**: Không cần handle retry logic, polling interval, websocket disconnect. Firebase SDK lo hết.
2. **Server nhẹ gánh**: Không phải chịu hàng nghìn request polling mỗi giây.

---

## 2. Firebase Data Structure

**Path**: `/user_jobs/{user_id}/{job_id}`

The Firebase path uses the `job_id` (PostgreSQL primary key) directly as the path segment. No mapping table or `firebase_ref_id` column is needed—the path is deterministic based on the job's primary key.

Data ở đây là **Ephemeral** (Tạm thời), có thể xóa sau khi Job hoàn thành 24h (dùng Cloud Functions để cleanup).

```json
{
  "status": "processing",
  "progress": 45,
  "step": "Upscaling...",
  "result": {
    "thumbnail_url": "https://s3.../thumb.jpg",
    "blurhash": "LEHV6nWB2yk8pyo0adR*.7kCMdnj",
    "media_url": null
  },
  "error": null,
  "created_at": 1703812345678,
  "updated_at": 1703812399999
}
```

### Field Descriptions

- `status`: `"pending" | "processing" | "completed" | "failed"` - Job status
- `progress`: `0-100` - Progress percentage (hiển thị thanh loading)
- `step`: `string` - Text hiển thị user đang làm gì (e.g., "Upscaling...", "Generating frames...")
- `result`: Object chứa kết quả khi có
  - `thumbnail_url`: URL preview (hiện ngay khi có)
  - `blurhash`: BlurHash string (load ngay lập tức)
  - `media_url`: Final media URL (có khi status = completed)
- `error`: `null | { message: string, code: string }` - Error info nếu failed
- `created_at`: Unix timestamp (milliseconds)
- `updated_at`: Unix timestamp (milliseconds)

---

## 3. Security Rules

Single Player nghĩa là User A không được thấy Job của User B.

```javascript
{
  "rules": {
    "user_jobs": {
      "$user_id": {
        // Chỉ user đó mới được read/write node của chính mình
        ".read": "auth != null && auth.uid == $user_id",
        ".write": "auth != null && auth.uid == $user_id"
        // Lưu ý: Backend (Admin SDK) luôn có quyền write, rule này chặn user khác hack.
      }
    }
  }
}
```

### Notes

- Backend sử dụng Admin SDK bypasses security rules (có full access)
- Client SDK chỉ có thể read/write path của chính user đó
- Firebase Auth UID phải match với `user_id` trong path

---

## 4. Workflow: The "Sync" Logic

Để đảm bảo data nhất quán giữa Postgres và Firebase:

### 4.1. Job Created

1. API insert vào Postgres `generation_jobs` với status `pending`
2. API set Firebase `/user_jobs/{uid}/{job_id}` = `{ status: 'pending', progress: 0, created_at: timestamp }`
3. API trả về `job_id` cho Client

### 4.2. Client Subscription

1. Client nhận `job_id`
2. Client subscribe: `firebase.database().ref('user_jobs/{uid}/{job_id}').on('value', snapshot => updateUI())`
3. Client hiển thị loading state ngay lập tức

### 4.3. Worker Processing

1. Worker pick up job từ queue
2. Worker update Postgres: `status = 'processing'`, `started_at = NOW()` (lifecycle state change only)
3. Worker sync Firebase: `{ status: 'processing', progress: 50, step: 'Upscaling...', updated_at: timestamp }` (progress stored exclusively in Firebase)
4. Client tự động nhận update qua Firebase subscription → Update UI

### 4.4. Job Completed

1. Worker upload S3 → Lấy URL
2. Worker tính toán `blurhash` từ media
3. Worker update Postgres:
   - `generation_media` record với `media_url`, `blurhash` (NOT NULL)
   - `generation_jobs` status = `completed`, `completed_at = NOW()` (lifecycle state change only)
4. Worker update Firebase: `{ status: 'completed', progress: 100, result: { media_url: '...', blurhash: '...' }, updated_at: timestamp }` (progress stored exclusively in Firebase)
5. Client nhận signal `completed` → Hiển thị ảnh/video dùng `blurhash` trước, sau đó load ảnh thật

### 4.5. Job Failed

1. Worker update Postgres: `status = 'failed'`, `error_message = '...'`, `error_code = '...'`
2. Worker update Firebase: `{ status: 'failed', error: { message: '...', code: '...' }, updated_at: timestamp }`
3. Client nhận signal `failed` → Hiển thị error message

---

## 5. Cleanup Strategy

Firebase data là ephemeral. Sau khi job completed/failed, có thể cleanup sau 24h.

### Implementation Options

1. **Cloud Functions**: Scheduled function chạy mỗi ngày, xóa các job đã completed/failed > 24h
2. **Worker cleanup**: Worker tự cleanup khi update status completed/failed (check `updated_at`)

### Cleanup Path

```
/user_jobs/{user_id}/{job_id} → DELETE if (status === 'completed' || status === 'failed') && (now - updated_at > 24h)
```

---

## 6. Integration với PostgreSQL

### Mapping Fields

| PostgreSQL (`generation_jobs`) | Firebase (`/user_jobs/{uid}/{job_id}`) |
|-------------------------------|----------------------------------------|
| `id` | `{job_id}` (path segment) |
| `status` | `status` |
| `error_message` | `error.message` |
| `error_code` | `error.code` |
| `started_at` | (derived from `created_at` + `updated_at`) |
| `completed_at` | (derived when `status === 'completed'`) |

**Note**: Progress (`progress: 0-100`, `step: "..."`) is stored exclusively in Firebase and never synced to Postgres. This eliminates write amplification from high-frequency progress updates.

### Sync Points

- **On Job Create**: Postgres INSERT → Firebase SET (initial state with `status: 'pending'`, `progress: 0`)
- **On Progress Update**: Firebase UPDATE `progress` only (no Postgres write)
- **On Job Complete**: Postgres UPDATE `status` + INSERT `generation_media` → Firebase UPDATE `status` + `result`
- **On Job Fail**: Postgres UPDATE `status` → Firebase UPDATE `status` + `error`

---

## 7. Error Handling

### Network Failures

- **Backend → Firebase**: Retry logic với exponential backoff. Nếu Firebase down, job vẫn hoàn thành trong Postgres, client có thể poll Postgres API fallback.
- **Client → Firebase**: Firebase SDK tự động reconnect. Nếu mất kết nối, client có thể fallback về polling Postgres API.

### Data Consistency

- Postgres là source of truth. Nếu Firebase data bị mất, có thể rebuild từ Postgres.
- Worker luôn update Postgres trước, sau đó sync Firebase (Postgres-first approach).

---

## 8. MVP Considerations

### Required

- Firebase Realtime Database setup
- Security rules cho user isolation
- Backend Admin SDK integration
- Client SDK subscription logic

### Optional (Future)

- Cloud Functions cho cleanup automation
- Firebase Analytics integration
- Offline support (Firebase SDK cache)

---

**Last Updated**: December 28, 2025

