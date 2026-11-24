# Functional Requirements (Target Specification)

- **System:** FinQuest: Pahlawan Finansial (Flutter Mobile App + Optional Laptop MySQL Sync API)
- **Source of truth:** `PRD.md` v6.0 & Mobile Programming (BBK4DBB3) P12–P15 deliverables.
- **Mode:** Offline-first (SQLite). Account login and cloud backup/restore are optional.

## Summary

| ID | Module | Connectivity | Summary |
| :--- | :--- | :--- | :--- |
| FR1 | Arena | Offline | Record saving transaction to attack monster and gain rewards |
| FR2 | Arena | Offline | Record expense by category with budget overage penalty |
| FR3 | Arena | Offline | Hero HP management, daily regeneration, and level-down at 0 HP |
| FR4 | Arena | Offline | Monster defeat victory bonus and adventure restart |
| FR5 | Budget | Offline | View daily category usage and edit daily spending limits |
| FR6 | Notification | Offline | Toggle 20:00 WIB daily reminder with Android 13+ permission flow |
| FR7 | History | Offline | View transaction history (latest 100) and transaction detail |
| FR8 | History | Offline | Edit existing transaction without recalculating past battle effects |
| FR9 | History | Offline | Delete transaction with confirmation dialog and *Urungkan* undo |
| FR10 | Quest | Online | Fetch business news from NewsAPI and open in-app reader sheet |
| FR11 | Quest | Online/Offline | Claim +50 attack buff after 10s countdown (max 3/day, cap +150) |
| FR12 | Account | Optional Online | Register, login, and logout against MySQL REST API with secure token |
| FR13 | Cloud Sync | Optional Online | Manual backup of local SQLite state & transactions to MySQL server |
| FR14 | Cloud Sync | Optional Online | Manual restore from MySQL server overwriting local SQLite data |

---

## 1. Battle Arena & Logging

### FR1 Record Saving (Attack Monster)
- **Requirement:** The user SHALL record a saving transaction in whole Rupiah without requiring an account or internet connection.
- **Acceptance criteria:**
  - Amount must be digits only and `> 0`; description is optional (≤ 100 chars, fallback *"Tabungan"*).
  - Double-tap is prevented via `_isSubmitting`.
  - Saving deals `max(1, amount ~/ 1000) + attackBuff` damage (0 if adventure complete), consumes `attackBuff` only on real damage, restores `max(1, amount ~/ 5000)` Hero HP (capped at 100), and adds Gold and EXP.
  - Transaction, character stats, and monster HP are committed atomically in one SQLite transaction.
- **Traceability:** NFR2, NFR7.

### FR2 Record Expense & Budget Penalty
- **Requirement:** The user SHALL record an expense under one of 5 fixed categories (`food`, `transport`, `entertainment`, `shopping`, `other`).
- **Acceptance criteria:**
  - `AttackForm` displays live daily usage (*"Terpakai hari ini: Rp X / Rp Y"*) and predicted HP loss if the typed amount exceeds the limit.
  - Only the newly exceeded amount (`GameRules.newOverage`) triggers penalties: `hpLost = max(1, newOverage ~/ 1000)` and `monsterHeal = max(1, newOverage ~/ 2000)` (0 if adventure complete).
  - Spending within or exactly equal to the limit causes zero penalty.
- **Traceability:** NFR2, NFR7.

### FR3 Hero HP, Daily Regeneration & Level-Down
- **Requirement:** The system SHALL manage Hero HP (`0..100`), regenerate `+25 HP` per elapsed calendar day, and trigger a level-down when HP reaches `0`.
- **Acceptance criteria:**
  - Daily rollover runs on app startup, app resume (`WidgetsBindingObserver`), and before logging transactions; backward device clock changes do nothing.
  - If HP reaches `0` at `level > 1`: `level -= 1`, `exp = (level - 1) * 100`, `hp = 100`.
  - If HP reaches `0` at `level == 1`: `level` stays `1`, `exp = 0`, `hp = 100`.
  - A non-dismissible Indonesian dialog informs the user.
- **Traceability:** NFR2, NFR7.

### FR4 Monster Victory & Adventure Restart
- **Requirement:** Defeating a monster SHALL grant victory rewards and advance to the next monster; defeating all 3 SHALL allow restarting the adventure.
- **Acceptance criteria:**
  - Finishing blow awards `+50 Gold` and `+50 EXP` in the same SQLite transaction and shows a non-dismissible victory dialog.
  - After Monster 3 (`ongkir`), an end-state card appears with **"Mulai Petualangan Baru"** while the transaction form remains usable.
- **Traceability:** NFR2, NFR4.

---

## 2. Daily Budget & Notifications

### FR5 Configure Daily Budgets
- **Requirement:** The user SHALL view daily spending progress per category and customize daily limits from Profile.
- **Acceptance criteria:**
  - Profile shows 5 category progress bars (`spentToday / dailyLimit`) and financial health label (*Sehat / Waspada / Boros*).
  - `BudgetSettingsSheet` validates each limit `>= Rp 1.000`.
  - Updated limits persist in SQLite immediately and apply to subsequent expenses.
- **Traceability:** NFR2, NFR4.

### FR6 Daily Reminder Notification & Permission
- **Requirement:** The user SHALL be able to enable or disable a local daily reminder at 20:00 WIB.
- **Acceptance criteria:**
  - Permission (`POST_NOTIFICATIONS` on Android 13+) is requested ONLY when the user toggles the switch ON, preceded by an explanation dialog.
  - If denied, the switch stays OFF, an Indonesian info message appears, and the app remains fully usable.
  - Schedule uses `Asia/Jakarta` timezone and `AndroidScheduleMode.inexactAllowWhileIdle`, surviving device reboots (`RECEIVE_BOOT_COMPLETED`).
- **Traceability:** NFR3, NFR6.

---

## 3. History & CRUD

### FR7 View Transaction History & Detail
- **Requirement:** The user SHALL browse the latest 100 transactions newest-first and open a detail view for any item.
- **Acceptance criteria:**
  - `HistoryScreen` uses `ListView.builder` with loading and empty states (*"Belum ada transaksi. Mulai catat di Arena!"*).
  - Tapping an item pushes `TransactionDetailScreen` showing description, formatted Rupiah, type, category (if expense), and Indonesian date/time.
- **Traceability:** NFR1, NFR4.

### FR8 Edit Transaction
- **Requirement:** The user SHALL edit a transaction's amount, description, and (for expenses) category.
- **Acceptance criteria:**
  - Type and date are read-only; notice explains battle effects are not recalculated.
  - Saving updates SQLite, refreshes History, Profile totals, and today's budget bars immediately via `GameStore`.
- **Traceability:** NFR2, NFR7.

### FR9 Delete Transaction with Confirmation & Undo
- **Requirement:** The user SHALL delete a transaction after confirming a dialog, with an option to undo immediately.
- **Acceptance criteria:**
  - Confirmation dialog warns that past battle effects are not rolled back.
  - Deleting pops back to History without throwing, updates totals/budgets, and shows a SnackBar with **"Urungkan"**.
  - Pressing **"Urungkan"** re-inserts the exact record into SQLite and restores totals/budgets.
- **Traceability:** NFR2, NFR4.

---

## 4. Daily Mission (NewsAPI)

### FR10 Fetch & Read Financial News
- **Requirement:** When online, the user SHALL browse business news headlines from NewsAPI and read article summaries in a bottom sheet.
- **Acceptance criteria:**
  - `NewsService.fetchFinancialNews()` is cached in `initState()` (never called in `build()`).
  - `[Removed]` articles are filtered out; images use `CachedNetworkImage`.
  - Offline or API errors (401, 429, timeout) display clear Indonesian messages with a **"Coba Lagi"** button without breaking other tabs.
- **Traceability:** NFR1, NFR3, NFR4.

### FR11 Claim Attack Buff
- **Requirement:** Reading an article for 10 seconds SHALL allow claiming a `+50` attack buff stored locally in SQLite.
- **Acceptance criteria:**
  - **"Klaim Buff"** button stays disabled for 10s with a live countdown (`Timer.periodic`, cancelled on dispose).
  - Enforces: unique URL claimed once ever, max 3 claims per local day, max active buff `+150`.
  - Claimed buff persists in SQLite and works offline in the Arena.
- **Traceability:** NFR2, NFR7.

---

## 5. Optional Account & Cloud Backup/Restore

### FR12 Optional Register, Login & Logout
- **Requirement:** Users MAY optionally register or log in from Profile to enable cloud backup/restore without losing guest access.
- **Acceptance criteria:**
  - Guest users can use FR1–FR11 without ever touching Login/Register.
  - Register (`display_name`, `email`, `password`) and Login (`email`, `password`) validate inputs inline and call the laptop MySQL REST API.
  - Received Bearer token is stored ONLY in `flutter_secure_storage`.
  - Logout calls `POST /api/v1/auth/logout`, deletes the local token, and returns Profile to guest state while **preserving local SQLite data**.
- **Traceability:** NFR3, NFR5.

### FR13 Manual Backup to MySQL Server
- **Requirement:** An authenticated user SHALL back up their entire local SQLite dataset to the MySQL server on demand.
- **Acceptance criteria:**
  - Pressing **"Cadangkan ke Server"** exports `game_state`, `monsters`, `budgets`, `transactions`, and `claimed_articles` as JSON and sends `POST /api/v1/sync/backup` with the Bearer token.
  - Displays loading indicator during upload and success timestamp or Indonesian error message on failure.
- **Traceability:** NFR3, NFR7.

### FR14 Manual Restore from MySQL Server
- **Requirement:** An authenticated user SHALL restore their latest cloud backup from the MySQL server onto the current device.
- **Acceptance criteria:**
  - Pressing **"Pulihkan dari Server"** shows a confirmation dialog warning that current local data on this device will be replaced.
  - On confirm, fetches `GET /api/v1/sync/restore`, replaces all SQLite tables inside a single `db.transaction(...)`, reloads `GameStore`, and updates all tabs immediately.
  - If no backup exists on the server (HTTP 404), shows *"Belum ada cadangan di server."* without modifying local SQLite data.
- **Traceability:** NFR2, NFR7.