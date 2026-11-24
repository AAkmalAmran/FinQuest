# API Contracts

- **Internal Backend (Optional Sync):** **Express.js + MySQL** REST API running on the team's laptop (`server/`).
- **Base URL:** Configured via `--dart-define=API_URL=...` (default `http://10.0.2.2:8000` for Android Emulator; use laptop Wi-Fi IP e.g. `http://192.168.1.10:8000` for physical phone).
- **Base Path:** `/api/v1`
- **Authentication:** JWT Bearer Token (`Authorization: Bearer <token>`) stored in `flutter_secure_storage`.
- **Format:** JSON only (`snake_case` keys). User-facing error messages are **Indonesian**.

---

## 1. Conventions

| Topic | Convention |
| :--- | :--- |
| Headers | `Content-Type: application/json`, `Accept: application/json`, and `Authorization: Bearer <token>` on protected routes |
| Timeout | 10 seconds per HTTP request |
| Money | Integer Rupiah (e.g. `50000`) |
| Dates | `yyyy-MM-dd` for date keys; ISO-8601 for timestamps |
| Error format | `{ "message": "Pesan bahasa Indonesia", "errors": { "field": ["..."] } }` |
| Status codes | `200` OK, `201` Created, `204` No Content, `401` Unauthorized, `404` Not Found, `422` Validation Error, `500` Server Error |

---

## 2. Authentication Endpoints

| Method | Path | Auth | Purpose |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/auth/register` | Guest | Create account & return JWT Bearer token |
| `POST` | `/api/v1/auth/login` | Guest | Verify credentials & return JWT Bearer token |
| `GET` | `/api/v1/auth/me` | Bearer | Get current user profile & `last_backed_up_at` |
| `POST` | `/api/v1/auth/logout` | Bearer | End session (`204`) |

### `POST /api/v1/auth/register`
Request:
```json
{
  "display_name": "Pahlawan Hemat",
  "email": "budi@student.telkomuniversity.ac.id",
  "password": "password123"
}
```
Response (`201 Created`):
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "id": 1,
    "display_name": "Pahlawan Hemat",
    "email": "budi@student.telkomuniversity.ac.id"
  }
}
```

### `POST /api/v1/auth/login`
Request:
```json
{
  "email": "budi@student.telkomuniversity.ac.id",
  "password": "password123"
}
```
Response (`200 OK`): Same structure as register. Invalid credentials return `401` with `{ "message": "Email atau kata sandi salah." }`.

---

## 3. Cloud Backup & Restore Endpoints

| Method | Path | Auth | Purpose |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/sync/backup` | Bearer | Overwrite user's cloud snapshot with local SQLite payload |
| `GET` | `/api/v1/sync/restore` | Bearer | Download user's latest cloud snapshot (`404` if none exists) |

### `POST /api/v1/sync/backup`
Request:
```json
{
  "schema_version": 1,
  "game_state": {
    "hero_name": "Pahlawan Hemat",
    "gold": 120,
    "exp": 240,
    "level": 3,
    "attack_buff": 50,
    "hp": 85,
    "max_hp": 100,
    "current_monster_id": "kopi",
    "is_adventure_complete": false,
    "last_regen_date": "2026-11-25",
    "reminder_enabled": true
  },
  "monsters": [
    { "id": "kopi", "current_hp": 180 },
    { "id": "belanja", "current_hp": 800 },
    { "id": "ongkir", "current_hp": 1500 }
  ],
  "budgets": {
    "food": 50000,
    "transport": 20000,
    "entertainment": 30000,
    "shopping": 50000,
    "other": 30000
  },
  "transactions": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "description": "Makan siang",
      "amount": 25000,
      "type": "expense",
      "category": "food",
      "date": "2026-11-25T12:30:00",
      "date_key": "2026-11-25"
    }
  ],
  "claimed_articles": [
    { "url": "https://example.com/news-1", "claimed_date": "2026-11-25" }
  ]
}
```
Response (`200 OK`):
```json
{
  "message": "Data berhasil dicadangkan ke server.",
  "transaction_count": 1,
  "backed_up_at": "2026-11-25T13:00:00+07:00"
}
```

### `GET /api/v1/sync/restore`
Response (`200 OK`):
```json
{
  "backed_up_at": "2026-11-25T13:00:00+07:00",
  "payload": {
    "schema_version": 1,
    "game_state": { "...": "..." },
    "monsters": [ "..." ],
    "budgets": { "...": "..." },
    "transactions": [ "..." ],
    "claimed_articles": [ "..." ]
  }
}
```
If no backup exists yet: `404 Not Found` with `{ "message": "Belum ada cadangan di server." }`.

**Security Rule:** Server MUST derive `user_id` strictly from the verified JWT Bearer token—never from request body fields.

---

## 4. External API (Quest — NewsAPI)

- **Endpoint:** `GET https://newsapi.org/v2/top-headlines?category=business&country=us&apiKey={NEWS_API_KEY}`
- **Key Injection:** `String.fromEnvironment('NEWS_API_KEY')` (never committed).
- **Error Mapping (Indonesian):**
  - Missing key → *"API key belum diatur. Jalankan aplikasi dengan --dart-define=NEWS_API_KEY=..."*
  - `401` → *"API key tidak valid. Periksa kembali key NewsAPI kamu."*
  - `429` → *"Kuota NewsAPI hari ini habis. Coba lagi besok."*
  - `SocketException` → *"Tidak ada koneksi internet. Misi harian butuh internet; fitur lain tetap bisa dipakai."*
  - `TimeoutException` → *"Koneksi terlalu lama. Coba lagi."*