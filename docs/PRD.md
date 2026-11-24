# Product Requirement Document (PRD) — v6.0 (SQLite Offline-First + Optional Express/MySQL Sync)

## FinQuest: Pahlawan Finansial (Mobile Application)

> **Revision summary (v5.2 → v6.0):**
>
> * **Offline-First Guest Mode (SQLite):** Replaced Firebase Auth/Firestore with local **SQLite (`sqflite`)** as the primary database. Users open the app directly into the Arena and use all core features offline without creating an account.
> * **Optional Cloud Backup/Restore (Express.js + MySQL):** Account registration/login is optional (accessed from Profile). Authenticated users store their Bearer token in `flutter_secure_storage` and trigger manual **Cadangkan (Backup)** and **Pulihkan (Restore)** to/from the team's local **Express.js + MySQL** server.
> * **Monorepo Layout:** Organized into a single repository (`FinQuest/`) containing `app/` (Flutter), `docs/` (PRD, Design, Requirements, Architecture, Team Division), and `server/` (Express.js REST API + MySQL).

---

## 1. Document Index

| Document | Scope |
| :--- | :--- |
| `README.md` | Root guide: how to run `app/` and `server/`, folder overview, team division |
| `docs/PRD.md` (this file) | Product overview, AI guardrails, monorepo structure, game math |
| `docs/DESIGN.md` | Brand palette (Forest/Sage/Gold/Cream), WCAG contrast, Material 3 `AppTheme` & `GameColors` |
| `docs/team-division.md` | Detailed 5-person role assignment, file ownership, Git branches, & weekly milestones |
| `docs/requirements/fr.md` | Functional requirements (`FR1–FR14`) and acceptance criteria |
| `docs/requirements/nfr.md` | Non-functional requirements (`NFR1–NFR9`), security/performance checklist, verification commands |
| `docs/architecture/erd.md` | Local SQLite schema (`finquest.db`) and Server MySQL schema |
| `docs/architecture/api-contracts.md` | Express REST API contracts (Auth, Backup/Restore) and NewsAPI integration |
| `docs/architecture/sequence-diagrams.md` | End-to-end Mermaid sequence diagrams |

---

## 2. Product Overview

* **App Name:** FinQuest: Pahlawan Finansial
* **Platform:** Flutter (Android primary) + Optional Express.js/MySQL local backend
* **Academic Context:** Pengembangan Aplikasi Bergerak (BBK4DBB3) Tugas Besar P13–P15.
* **Target Users & Problem:** University students and young adults (18–25) who struggle to log daily expenses consistently and stick to spending limits.
* **Core Concept (*Budget Survival*):** Saving damages "Monster Boros" and heals the hero; spending **beyond self-set daily limits** hurts the hero (level-down at 0 HP) and heals the monster; reading finance news grants attack buffs.

### Core Features (MVP — 5 Features)

| # | Feature | Mode | Summary |
| :--- | :--- | :--- | :--- |
| F1 | **Arena (Log + Battle)** | Offline (SQLite) | Log savings/expenses by category; monster battle; hero HP & level-down |
| F2 | **Daily Budget & Reminder** | Offline (SQLite) | Per-category daily limits, progress bars, +25 HP/day regen, 20:00 WIB reminder |
| F3 | **History & Full CRUD** | Offline (SQLite) | List, detail, edit, and delete (with confirmation + *Urungkan* undo) |
| F4 | **Daily Mission (News)** | Online (NewsAPI) | Read up to 3 business articles/day (10s timer) for +50 attack buff |
| F5 | **Account & Cloud Sync** | Optional Online (Express + MySQL) | Optional register/login in Profile; manual Backup & Restore across devices |

### Low-Fidelity Wireframes

```text
ARENA (Default Start Screen)        RIWAYAT                       PROFIL (Includes Optional Sync)
┌────────────────────────┐    ┌────────────────────────┐    ┌────────────────────────┐
│ Pahlawan Hemat  Lv 3   │    │ Riwayat Transaksi      │    │ Stats: Lv, HP, Gold    │
│ HP Hero ████████░░ 80  │    │ ┌────────────────────┐ │    │ Anggaran Harian (bars) │
│ ┌────────────────────┐ │    │ │↓ Makan siang       │ │    │ [Atur Batas Harian]    │
│ │   (monster image)  │ │    │ │  Makan  -Rp 25.000 │ │    │ Pengingat Harian  [⏻]  │
│ │ Monster Kopi Susu  │ │    │ ├────────────────────┤ │    │ ┌────────────────────┐ │
│ │ HP ██████░░░ 210   │ │    │ │↑ Tabungan          │ │    │ │ Akun & Cadangan    │ │
│ └────────────────────┘ │    │ │         +Rp 50.000 │ │    │ │ [Masuk / Daftar]   │ │
│ [Nominal Transaksi   ] │    │ └────────────────────┘ │    │ │ (or if logged in:) │ │
│ [Kategori ▼ Makan    ] │    │ (tap → Detail/Edit/Del)│    │ │ [Cadangkan]        │ │
│ Terpakai Rp 30k/50k    │    └────────────────────────┘    │ │ [Pulihkan] [Keluar]│ │
│ [Deskripsi           ] │                                  │ └────────────────────┘ │
│ [Catat Tabungan]       │    [Arena|Riwayat|Quest|Profil]  └────────────────────────┘
│ [Catat Pengeluaran]    │
└────────────────────────┘
```

---

## 3. Strict Guardrails for AI Coding Assistants

1. **State Management (`ChangeNotifier` + `provider` ONLY — P12 architecture):** Use a single `GameStore extends ChangeNotifier` (`app/lib/state/game_store.dart`). FORBIDDEN: `bloc`, `riverpod`, `get`, `mobx`, extra `ChangeNotifier` classes, or global mutable state. Read reactively via `context.select<GameStore, T>(...)` or `Selector` in `build()`; call actions via `context.read<GameStore>().method()` in callbacks. Keep local UI state (`TextEditingController`, `_isSubmitting`, tab index, `Timer`) in `StatefulWidget` + `setState()`.
2. **Storage & Backend Boundaries:**
   * Primary local DB: **`sqflite` + `path`** (`app/lib/services/database_helper.dart`).
   * Optional remote sync & auth: **`http`** (`app/lib/services/auth_api_service.dart`, `app/lib/services/sync_api_service.dart`) + **`flutter_secure_storage`** (`app/lib/services/token_storage.dart`) talking to the **Express.js + MySQL** backend in `server/`.
   * FORBIDDEN: `firebase_*`, `cloud_firestore`, `hive`, `isar`, `drift`, `shared_preferences`.
   * Only `app/lib/services/*.dart` may import `sqflite`, `http`, `flutter_secure_storage`, `flutter_local_notifications`, or `timezone`. **Only `GameStore`** calls `DatabaseHelper`, `SyncApiService`, `AuthApiService`, and `NotificationService`.
3. **Atomic Write-Through (SQLite):** Every multi-table mutation (e.g., saving + monster HP + character stats) MUST run inside `db.transaction(...)`. Update SQLite first, then update in-memory state in `GameStore` (copy-on-change) and call `notifyListeners()`.
4. **Locked Dependencies:**
   * **Flutter (`app/pubspec.yaml`):** `flutter`, `provider`, `sqflite`, `path`, `http`, `intl`, `flutter_secure_storage`, `flutter_local_notifications`, `timezone`, `cached_network_image`.
   * **Express Server (`server/package.json`):** `express`, `mysql2`, `bcryptjs`, `jsonwebtoken`, `dotenv`, `cors`.
5. **Single Source of Theme & Game Math:**
   * All colors, light/dark `ThemeData`, and `GameColors` `ThemeExtension` live ONLY in `app/lib/core/theme/app_theme.dart` (per `docs/DESIGN.md`). Never hardcode `Color(0x...)` or `Colors.xyz` in widgets.
   * All game formulas live ONLY in `app/lib/core/constants/game_rules.dart`.
6. **Layout & Language:** Wrap every screen in `ResponsiveBody` (max width 600, centered). Every form screen/sheet must be scrollable (`SingleChildScrollView`; bottom sheets use `isScrollControlled: true` + `viewInsets.bottom`). Keep widgets ≤ ~150 lines. Code, comments, enums, and DB/JSON keys are **English**; all UI strings and API error messages are **Indonesian**.
7. **Secrets & Server URL:** Never hardcode API keys or server IPs. Inject via `--dart-define`:
   * `API_URL` (default `http://10.0.2.2:8000`)
   * `NEWS_API_KEY` (required for Quest)

### Install & Run Commands (Monorepo)

```bash
# 1. Run Express.js + MySQL backend (optional sync server)
cd server
npm install
cp .env.example .env   # fill DB_HOST, DB_USER, DB_PASS, DB_NAME, JWT_SECRET
npm run dev            # listens on port 8000

# 2. Install Flutter dependencies & run app (Android emulator)
cd ../app
flutter pub add provider sqflite path http intl flutter_secure_storage flutter_local_notifications timezone cached_network_image
flutter run \
  --dart-define=API_URL=http://10.0.2.2:8000 \
  --dart-define=NEWS_API_KEY=YOUR_KEY
```

---

## 4. Project Architecture & Directory Tree (Monorepo)

Layered P12 architecture inside `app/`: **UI (`screens/`, `widgets/`) → Store (`state/game_store.dart`) → Local DB & Remote Services (`services/`)**.

```text
FinQuest/
├── README.md                               # Root guide: setup app & server, structure, task division
├── .gitignore                              # Global ignores (.vscode/, .DS_Store, .env)
├── docs/
│   ├── PRD.md                              # Master product & technical guardrails
│   ├── DESIGN.md                           # Visual design system & WCAG contrast rules
│   ├── team-division.md                    # 5-person task division, file ownership & branches
│   ├── requirements/
│   │   ├── fr.md                           # Functional requirements (FR1-FR14)
│   │   └── nfr.md                          # Non-functional requirements (NFR1-NFR9)
│   └── architecture/
│       ├── erd.md                          # SQLite & MySQL schemas
│       ├── api-contracts.md                # Express REST API & NewsAPI contracts
│       └── sequence-diagrams.md            # Mermaid execution flows
├── server/                                 # Optional Laptop REST API (Express.js + MySQL)
│   ├── package.json
│   ├── .env.example                        # PORT, DB_*, JWT_SECRET (never commit .env)
│   ├── .gitignore                          # node_modules/, .env
│   ├── database/
│   │   └── schema.sql                      # CREATE TABLE users & user_backups
│   └── src/
│       ├── index.js                        # Express app bootstrap & middleware
│       ├── config/
│       │   └── db.js                       # mysql2 connection pool
│       ├── middleware/
│       │   └── auth.middleware.js          # JWT Bearer verification
│       ├── routes/
│       │   ├── auth.routes.js              # /api/v1/auth/*
│       │   └── sync.routes.js              # /api/v1/sync/*
│       └── controllers/
│           ├── auth.controller.js
│           └── sync.controller.js
└── app/                                    # Flutter Mobile Application
    ├── android/
    ├── ios/
    ├── pubspec.yaml
    ├── .gitignore                          # Flutter ignores (build/, .dart_tool/, key.properties)
    ├── assets/images/
    │   ├── monster_kopi.png
    │   ├── monster_belanja.png
    │   └── monster_ongkir.png
    ├── lib/
    │   ├── main.dart
    │   ├── core/
    │   │   ├── constants/
    │   │   │   ├── api_endpoints.dart      # API_URL & NEWS_API_KEY via --dart-define
    │   │   │   └── game_rules.dart         # Constants + pure game math
    │   │   ├── theme/
    │   │   │   └── app_theme.dart          # AppColors, AppTheme (light/dark), GameColors
    │   │   └── utils/
    │   │       ├── formatters.dart         # formatRupiah, formatDate, dateKey, todayKey
    │   │       └── validators.dart         # Indonesian form validators
    │   ├── models/
    │   │   ├── monster.dart
    │   │   ├── character.dart
    │   │   ├── transaction_record.dart
    │   │   ├── article.dart
    │   │   ├── budget.dart
    │   │   ├── game_results.dart
    │   │   └── user_session.dart           # Optional logged-in user info (id, displayName, email)
    │   ├── services/
    │   │   ├── database_helper.dart        # SQLite (sqflite) CRUD, atomic transactions, export/import
    │   │   ├── token_storage.dart          # flutter_secure_storage wrapper for Bearer token
    │   │   ├── auth_api_service.dart       # Express API register, login, logout, me
    │   │   ├── sync_api_service.dart       # Express API backup (POST) & restore (GET)
    │   │   ├── news_service.dart           # NewsAPI HTTP GET
    │   │   └── notification_service.dart   # Daily 20:00 WIB local reminder
    │   ├── state/
    │   │   └── game_store.dart             # Single ChangeNotifier store
    │   ├── screens/
    │   │   ├── main_navigation_screen.dart # App entry (no forced login), 4-tab NavigationBar
    │   │   ├── arena/
    │   │   │   ├── arena_screen.dart
    │   │   │   └── widgets/
    │   │   │       ├── hero_status.dart
    │   │   │       ├── monster_display.dart
    │   │   │       └── attack_form.dart
    │   │   ├── history/
    │   │   │   ├── history_screen.dart
    │   │   │   ├── transaction_detail_screen.dart
    │   │   │   └── transaction_edit_screen.dart
    │   │   ├── quest/
    │   │   │   ├── quest_screen.dart
    │   │   │   └── widgets/
    │   │   │       ├── news_card.dart
    │   │   │       └── article_reader_sheet.dart
    │   │   └── profile/
    │   │       ├── profile_screen.dart
    │   │       ├── login_screen.dart       # Pushed from Profile (optional)
    │   │       ├── register_screen.dart    # Pushed from Login (optional)
    │   │       └── widgets/
    │   │           ├── budget_settings_sheet.dart
    │   │           └── cloud_sync_card.dart
    │   └── widgets/
    │       ├── hp_bar.dart
    │       └── responsive_body.dart
    └── test/
        ├── game_rules_test.dart
        └── game_store_test.dart
```

---

## 5. Core Game Rules (`app/lib/core/constants/game_rules.dart`)

All game math runs locally in Dart using integer Rupiah:

* **Monsters (fixed order):** `kopi` (*Monster Kopi Susu Mahal*, 300 HP) → `belanja` (*Monster Belanja Impulsif*, 800 HP) → `ongkir` (*Monster Ongkir Makanan*, 1500 HP).
* **Saving ("Catat Tabungan"):**
  * `baseDamage = max(1, amount ~/ 1000)`; `damage = isAdventureComplete ? 0 : baseDamage + attackBuff` (buff resets to 0 after real attack).
  * `gold += amount ~/ 1000`; `exp += 10 + (amount ~/ 5000)`; `level = 1 + exp ~/ 100`.
  * `heroHp = min(100, heroHp + max(1, amount ~/ 5000))`.
  * Defeating a monster grants `+50 Gold` and `+50 EXP`.
* **Expense ("Catat Pengeluaran") — *New Overage Only*:**
  * Default daily limits: `food: 50000`, `transport: 20000`, `entertainment: 30000`, `shopping: 50000`, `other: 30000` (min `1000`).
  * `newOverage = max(0, spentBefore + amount - limit) - max(0, spentBefore - limit)`.
  * If `newOverage == 0`: no penalty.
  * If `newOverage > 0`: `hpLost = max(1, newOverage ~/ 1000)`; `monsterHeal = isAdventureComplete ? 0 : max(1, newOverage ~/ 2000)`.
* **Hero HP Depletion (`hp <= 0`):** If `level > 1`, `level -= 1` and `exp = (level - 1) * 100`; if `level == 1`, `exp = 0`. Refill `hp = 100`.
* **Daily Rollover:** On new calendar day (`todayKey()`), regenerate `+25 HP` per elapsed day (capped at 100) and reset daily spent counters.
* **Daily Mission (Quest):** Max 3 articles/day, 10s reading countdown, `+50` buff per unique URL, max buff `+150`.
* **Financial Health Ratio:** `totalSaving / (totalSaving + totalExpense)` → `>= 0.70` *Sehat*, `0.40–0.69` *Waspada*, `< 0.40` *Boros*.

---

## 6. AI Prompting Template (For Teammates)

```text
Context & Rules:
- Project: FinQuest monorepo (`app/` Flutter, `docs/` specs, `server/` Express.js + MySQL).
- Architecture: SQLite offline-first via `sqflite`, optional Express/MySQL REST sync via `http` + `flutter_secure_storage`, state management via `ChangeNotifier` + `provider`.
- Obey `docs/PRD.md`, `docs/DESIGN.md`, `docs/team-division.md`, `docs/requirements/fr.md`, `docs/architecture/erd.md`, and `docs/architecture/api-contracts.md`.
- STATE: Single `GameStore extends ChangeNotifier` (`app/lib/state/game_store.dart`). Use `context.select`/`Selector` in `build()`, `context.read` in callbacks. No Bloc/Riverpod/GetX/Firebase.
- BOUNDARIES: Only `app/lib/services/*` may import sqflite, http, flutter_secure_storage, or notifications. UI only talks to `GameStore`.
- THEME: Use `Theme.of(context).colorScheme` and `GameColors` from `app/lib/core/theme/app_theme.dart` only. No hardcoded colors. Support light & dark mode.
- LAYOUT: Wrap screens in `ResponsiveBody`; keep widgets <= 150 lines; Indonesian UI strings, English code/DB/JSON keys.
- Modify ONLY this file: [INSERT_FILE_PATH, e.g., app/lib/state/game_store.dart]
```

---

## 7. Team Division Summary

Detailed task ownership, file boundaries, and Git workflow for all **5 team members** are documented in `docs/team-division.md`.