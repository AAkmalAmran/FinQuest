# Sequence Diagrams

Target execution flows for FinQuest v6.0 (**SQLite Offline-First** + **Optional MySQL REST Sync**).

## 1. App Startup (Offline-First, No Login Required)

```mermaid
sequenceDiagram
    autonumber
    participant App as main.dart
    participant Store as GameStore
    participant DB as SQLite (DatabaseHelper)
    participant Sec as TokenStorage
    participant UI as MainNavigationScreen

    App->>Store: Create GameStore & call init()
    Store->>DB: Open finquest.db (seed defaults if first run)
    DB-->>Store: game_state, monsters, budgets, recent transactions, claimed_articles
    Store->>Store: checkDailyRollover() (+25 HP/day if date changed)
    Store->>Sec: readToken() (optional cached session)
    Sec-->>Store: token or null
    Store-->>UI: notifyListeners() → Render Arena immediately
```

## 2. Record Expense & Budget Survival (100% Offline)

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant Form as AttackForm
    participant Store as GameStore
    participant Rules as GameRules
    participant DB as SQLite

    U->>Form: Enter amount, category, description & tap "Catat Pengeluaran"
    Form->>Store: recordExpense(amount, desc, category)
    Store->>Store: checkDailyRollover()
    Store->>Rules: newOverage(limit, spentBefore, amount)
    Rules-->>Store: newOverage, hpLost, monsterHeal
    Store->>Store: Apply HP loss (if <= 0, trigger level-down & refill HP)
    Store->>DB: BEGIN transaction (INSERT transaction, UPDATE game_state & monster) COMMIT
    DB-->>Store: Committed
    Store-->>Form: notifyListeners() & return ExpenseResult
    Form->>U: Show SnackBar (+ Level-Down Dialog if hpDepleted)
```

## 3. Transaction Delete & Undo (100% Offline)

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant Detail as TransactionDetailScreen
    participant Hist as HistoryScreen
    participant Store as GameStore
    participant DB as SQLite

    U->>Detail: Tap "Hapus" → Confirm Dialog
    Detail->>Store: deleteTransaction(record)
    Store->>DB: DELETE FROM transactions WHERE id = ?
    DB-->>Store: Deleted
    Store-->>Detail: notifyListeners()
    Detail->>Hist: Navigator.pop('deleted')
    Hist->>U: SnackBar "Transaksi dihapus" [Urungkan]
    opt User taps Urungkan
        U->>Hist: Tap "Urungkan"
        Hist->>Store: restoreTransaction(record)
        Store->>DB: INSERT INTO transactions (...)
        DB-->>Store: Restored
        Store-->>Hist: notifyListeners()
    end
```

## 4. Daily Mission — Read News & Claim Buff

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant Quest as QuestScreen
    participant News as NewsService
    participant Sheet as ArticleReaderSheet
    participant Store as GameStore
    participant DB as SQLite

    Quest->>News: fetchFinancialNews() (cached in initState)
    News-->>Quest: List<Article> (or NewsException if offline)
    U->>Quest: Tap NewsCard
    Quest->>Sheet: Open modal sheet & start 10s countdown
    U->>Sheet: Wait 10s & tap "Klaim Buff"
    Sheet->>Store: claimArticle(article)
    Store->>Store: Validate URL not claimed, claimsToday < 3, buff < 150
    Store->>DB: BEGIN transaction (INSERT claimed_articles, UPDATE attack_buff += 50) COMMIT
    DB-->>Store: Committed
    Store-->>Quest: notifyListeners() & return ArticleClaimResult
```

## 5. Optional Login & Manual Cloud Backup (Laptop MySQL)

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant Prof as Profile / LoginScreen
    participant Store as GameStore
    participant Auth as AuthApiService
    participant Sec as TokenStorage
    participant Sync as SyncApiService
    participant DB as SQLite
    participant API as Laptop REST API + MySQL

    U->>Prof: Login (email, password)
    Prof->>Store: login(email, password)
    Store->>Auth: POST /api/v1/auth/login
    Auth->>API: Verify credentials
    API-->>Auth: 200 { token, user }
    Store->>Sec: saveToken(token)
    Store-->>Prof: notifyListeners() (show Cloud Sync buttons)

    U->>Prof: Tap "Cadangkan ke Server"
    Prof->>Store: backupToCloud()
    Store->>DB: exportAllTablesAsJson()
    DB-->>Store: JSON payload
    Store->>Sync: POST /api/v1/sync/backup (Bearer token + payload)
    Sync->>API: Upsert user_backups in MySQL
    API-->>Sync: 200 { backed_up_at, transaction_count }
    Sync-->>Store: Success
    Store-->>Prof: Show SnackBar "Data berhasil dicadangkan"
```

## 6. Manual Cloud Restore (New Device / After Reinstall)

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant Prof as ProfileScreen
    participant Store as GameStore
    participant Sync as SyncApiService
    participant API as Laptop REST API + MySQL
    participant DB as SQLite

    U->>Prof: Tap "Pulihkan dari Server" → Confirm Dialog
    Prof->>Store: restoreFromCloud()
    Store->>Sync: GET /api/v1/sync/restore (Bearer token)
    Sync->>API: Query user_backups by token's user_id
    alt No backup found
        API-->>Sync: 404
        Sync-->>Store: Throw "Belum ada cadangan di server."
    else Backup exists
        API-->>Sync: 200 { payload }
        Store->>DB: BEGIN transaction (CLEAR & INSERT all tables from payload) COMMIT
        DB-->>Store: Restored
        Store->>Store: Reload in-memory state from SQLite
        Store-->>Prof: notifyListeners() → All tabs updated
    end
```