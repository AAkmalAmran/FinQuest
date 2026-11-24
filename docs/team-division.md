# Team Division (5 Members) — FinQuest

This document defines a balanced 5-person task split for the **FinQuest** monorepo (`app/`, `docs/`, `server/`). Every team member owns **both technical/logic responsibilities and specific UI screens**, with strict file ownership to prevent Git merge conflicts.

---

## 1. Role, Screen Ownership & Git Branch Summary

| Member | Primary Role | UI Screens / Components Owned | Git Branch |
| :--- | :--- | :--- | :--- |
| **Member 1** | **Backend, Database, Sync, Auth UI & Release Lead** | `LoginScreen`, `RegisterScreen`, `CloudSyncCard` | `feat/server-auth-release` |
| **Member 2** | **Game Mechanics, External API & Quest UI Lead** | `QuestScreen`, `NewsCard`, `ArticleReaderSheet` | `feat/game-logic-quest` |
| **Member 3** | **Design System, Navigation Shell & Arena UI Lead** | `MainNavigationScreen`, `ArenaScreen`, `HeroStatus`, `MonsterDisplay`, `AttackForm`, `HpBar`, `ResponsiveBody` | `feat/theme-arena-ui` |
| **Member 4** | **History & Full CRUD UI Lead** | `HistoryScreen`, `TransactionDetailScreen`, `TransactionEditScreen` | `feat/history-crud-ui` |
| **Member 5** | **Profile, Daily Budget UI & Notifications Lead** | `ProfileScreen`, `BudgetSettingsSheet` | `feat/profile-budget-notif` |

---

## 2. Detailed Responsibilities & File Ownership

### Member 1 — Backend (Express.js + MySQL), SQLite, Sync, Auth UI & Release
Owns the Express.js + MySQL server, local SQLite persistence, secure authentication, cloud backup/restore, the optional Auth UI screens, and final release packaging.

* **Screens / UI Built:**
  * **`LoginScreen` (`app/lib/screens/profile/login_screen.dart`):** Email & password form with inline validation, visibility toggle, loading spinner, and link to Register.
  * **`RegisterScreen` (`app/lib/screens/profile/register_screen.dart`):** Hero Name, Email, Password, and Confirm Password form.
  * **`CloudSyncCard` (`app/lib/screens/profile/widgets/cloud_sync_card.dart`):** Embedded card in Profile showing guest *"Masuk / Daftar"* CTA or authenticated account info with **"Cadangkan ke Server"**, **"Pulihkan dari Server"** (with overwrite confirmation dialog), and **"Keluar"** buttons.
* **Backend, Data & Release Scope:**
  * Build the **Express.js + MySQL** REST API (`server/`) per `docs/architecture/api-contracts.md` (`register`, `login`, `me`, `logout`, `sync/backup`, `sync/restore`) using `bcryptjs` and JWT Bearer tokens.
  * Write `server/database/schema.sql` (`users` and `user_backups` tables).
  * Build `DatabaseHelper` (`sqflite`) with atomic `db.transaction(...)` writes, default seeding, and JSON export/import.
  * Implement `TokenStorage` (`flutter_secure_storage`), `AuthApiService`, and `SyncApiService`.
  * Configure Android release signing, `--obfuscate --split-debug-info` APK/AAB build, launcher icon, minimal manifest permissions, and root `README.md`.
* **File Ownership:**
  * `server/**`
  * `app/lib/models/user_session.dart`
  * `app/lib/services/{database_helper,token_storage,auth_api_service,sync_api_service}.dart`
  * `app/lib/screens/profile/{login_screen,register_screen}.dart`
  * `app/lib/screens/profile/widgets/cloud_sync_card.dart`
  * `app/android/**` (signing, Gradle, manifest)
  * `README.md`
* **FR / NFR Coverage:** `FR12`, `FR13`, `FR14`, `NFR3`, `NFR9`, plus offline persistence for `FR1–FR9`.

---

### Member 2 — Game Mechanics, External API (NewsAPI) & Quest UI
Owns the RPG battle/budget math, centralized state management (`GameStore`), external NewsAPI integration, unit tests, and the entire **Quest** tab UI.

* **Screens / UI Built:**
  * **`QuestScreen` (`app/lib/screens/quest/quest_screen.dart`):** Daily Mission tab displaying daily quota header (`0/3 artikel`), active buff chip, `FutureBuilder` list (loading, empty, and offline/API error states with **"Coba Lagi"**).
  * **`NewsCard` (`app/lib/screens/quest/widgets/news_card.dart`):** Article card using `CachedNetworkImage` (`memCacheHeight: 180`), title, source, formatted date, and *"Sudah diklaim"* badge.
  * **`ArticleReaderSheet` (`app/lib/screens/quest/widgets/article_reader_sheet.dart`):** Modal bottom sheet showing article summary and a 10-second `Timer.periodic` countdown before enabling **"Klaim Buff"**.
* **Logic & API Scope:**
  * Implement all pure RPG and Budget Survival formulas in `game_rules.dart` (saving damage, `newOverage`, hero HP loss, monster heal, level-down at 0 HP, +25 HP daily regen, victory bonus).
  * Create core Dart models (`monster`, `character`, `transaction_record`, `article`, `budget`, `game_results`).
  * Implement **`GameStore extends ChangeNotifier`** (`app/lib/state/game_store.dart`) with `context.select`-friendly copy-on-change publishing and `checkDailyRollover()`.
  * Implement `api_endpoints.dart` and `news_service.dart` (NewsAPI HTTP GET + Indonesian error mapping).
  * Write automated unit tests (`game_rules_test.dart`, `game_store_test.dart`).
* **File Ownership:**
  * `app/lib/core/constants/{game_rules,api_endpoints}.dart`
  * `app/lib/models/{monster,character,transaction_record,article,budget,game_results}.dart`
  * `app/lib/services/news_service.dart`
  * `app/lib/state/game_store.dart`
  * `app/lib/screens/quest/**`
  * `app/test/{game_rules_test,game_store_test}.dart`
* **FR / NFR Coverage:** Core logic for `FR1–FR4`, full `FR10–FR11`, `NFR7`.

---

### Member 3 — Design System, Navigation Shell & Arena UI
Owns visual consistency (`docs/DESIGN.md`), shared UI components, app entry/navigation, and the entire **Arena** tab UI.

* **Screens / UI Built:**
  * **`MainNavigationScreen` (`app/lib/screens/main_navigation_screen.dart`):** Material 3 4-tab `NavigationBar` (`Arena`, `Riwayat`, `Quest`, `Profil`) with `IndexedStack` and `WidgetsBindingObserver` triggering daily rollover on app resume.
  * **`ArenaScreen` (`app/lib/screens/arena/arena_screen.dart`):** Main battle screen combining hero stats, monster display, end-of-adventure restart card, and transaction form, plus Victory and Level-Down dialogs.
  * **`HeroStatus`, `MonsterDisplay`, `AttackForm` (`app/lib/screens/arena/widgets/*`):** Hero level/HP/buff display, monster artwork & HP display, and interactive transaction form with live daily budget usage hint and predicted HP loss warning.
  * **Shared Widgets (`HpBar`, `ResponsiveBody`):** Animated HP bar and 600 dp max-width centered responsive wrapper.
* **Theme & Utilities Scope:**
  * Implement `app_theme.dart` (`AppColors`, `AppTheme.light`, `AppTheme.dark`, and `GameColors` `ThemeExtension`) strictly following WCAG contrast rules in `docs/DESIGN.md`.
  * Build `formatters.dart` (`formatRupiah`, `formatDate`, `dateKey`, `todayKey`) and `validators.dart`.
  * Wire `main.dart` and commit monster image assets (`app/assets/images/`).
* **File Ownership:**
  * `app/lib/main.dart`
  * `app/lib/core/theme/app_theme.dart`
  * `app/lib/core/utils/{formatters,validators}.dart`
  * `app/lib/widgets/{hp_bar,responsive_body}.dart`
  * `app/lib/screens/main_navigation_screen.dart`
  * `app/lib/screens/arena/**`
  * `app/assets/images/**`
* **FR / NFR Coverage:** UI for `FR1–FR4`, `NFR4`, `NFR8`.

---

### Member 4 — History & Full Transaction CRUD UI
Owns the complete transaction history browsing, detail inspection, editing, and deletion-with-undo user experience.

* **Screens / UI Built:**
  * **`HistoryScreen` (`app/lib/screens/history/history_screen.dart`):** Latest 100 transactions rendered via `ListView.builder` (newest first), loading & empty states (*"Belum ada transaksi. Mulai catat di Arena!"*), and **"Urungkan"** (Undo) SnackBar after deletion.
  * **`TransactionDetailScreen` (`app/lib/screens/history/transaction_detail_screen.dart`):** Displays full transaction details, **"Ubah"** button, and **"Hapus"** button with confirmation dialog (safe fallback while popping after deletion).
  * **`TransactionEditScreen` (`app/lib/screens/history/transaction_edit_screen.dart`):** Scrollable edit form for amount, description, and category (expenses only), with read-only type/date and notice that battle effects are not recalculated.
* **Quality Scope:**
  * Verify real-time UI reactivity across History and Profile totals via `context.select` after edit/delete/undo.
  * Perform keyboard-open overflow testing on small phones (360×640) and landscape mode across CRUD forms.
* **File Ownership:**
  * `app/lib/screens/history/**`
* **FR / NFR Coverage:** `FR7`, `FR8`, `FR9`, `NFR1`.

---

### Member 5 — Profile, Daily Budget UI & Local Notifications
Owns the **Profile** tab, daily budget configuration UI, 20:00 WIB local reminder notifications (including Android 13+ permission flow), and cross-device QA.

* **Screens / UI Built:**
  * **`ProfileScreen` (`app/lib/screens/profile/profile_screen.dart`):** Hero stats summary card, per-category Daily Budget progress bars, Financial Health ratio card (*Sehat / Waspada / Boros*), Daily Reminder `SwitchListTile` (with pre-permission explanation dialog), and layout slot embedding Member 1's `CloudSyncCard`.
  * **`BudgetSettingsSheet` (`app/lib/screens/profile/widgets/budget_settings_sheet.dart`):** Scrollable modal bottom sheet (`isScrollControlled: true` + `viewInsets.bottom`) to edit daily limits for all 5 categories with `>= Rp 1.000` validation.
* **Notification & QA Scope:**
  * Implement `NotificationService` (`flutter_local_notifications` + `timezone` `Asia/Jakarta` at 20:00 WIB, `AndroidScheduleMode.inexactAllowWhileIdle`, Android 13+ `POST_NOTIFICATIONS` runtime permission flow).
  * Coordinate notification manifest receivers & core library desugaring with Member 1.
  * Lead Light/Dark mode, airplane-mode offline reliability, and `flutter run --profile` jank verification across phone and tablet emulators.
* **File Ownership:**
  * `app/lib/services/notification_service.dart`
  * `app/lib/screens/profile/profile_screen.dart`
  * `app/lib/screens/profile/widgets/budget_settings_sheet.dart`
* **FR / NFR Coverage:** `FR5`, `FR6`, `NFR2`, `NFR6`.

---

## 3. Integration Order & Git Workflow

1. **Phase 1 — Foundation & P13 Deliverable (Week 1):**
   * **Member 2** & **Member 3** merge core models, `game_rules.dart`, `app_theme.dart`, and `ResponsiveBody` to `main` first.
   * All 5 members build their assigned screens with sample/placeholder state so every tab and route is navigable for the P13 review.
2. **Phase 2 — Core Features & P14 Deliverable (Weeks 2–4):**
   * **Member 1** finishes `server/` (Express + MySQL), `database_helper.dart`, Auth/Sync services, and Auth UI.
   * **Member 2** wires `GameStore` to `DatabaseHelper` and `NewsService`, and completes the Quest tab.
   * **Members 3, 4, 5** wire Arena, History CRUD, Profile/Budgets, and Local Notifications to `GameStore`.
3. **Phase 3 — QA, Release & P15 Deliverable (Weeks 5–6):**
   * Full end-to-end integration test (offline battle/CRUD + online Quest + optional Login → Backup → Reinstall/Clear → Restore).
   * **Member 5** runs QA & profile-mode checks; **Member 1** produces the signed obfuscated release APK/AAB and finalizes `README.md`.

### Daily Git Rules
* Work on your assigned feature branch and open a Pull Request to `main` only after `flutter analyze` reports **0 errors and 0 warnings**.
* Use scoped commit messages: `feat(app): ...`, `feat(server): ...`, `fix(app): ...`, or `docs: ...`.
* Do not edit files outside your ownership list without coordinating with the owner first (especially `game_store.dart` and `pubspec.yaml`).