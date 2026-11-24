# Non-Functional Requirements

- **Source:** FinQuest PRD v6.0 & Mobile Programming (BBK4DBB3) P13–P15 quality rubrics.
- **Status:** Target specification for SQLite offline-first + optional MySQL REST sync.

| ID | Characteristic | Target | Verification |
| :--- | :--- | :--- | :--- |
| NFR1 | Performance efficiency | Smooth 60 fps scrolling in History & Quest; `ListView.builder`; `CachedNetworkImage` with `memCacheHeight: 180`; scoped rebuilds via `context.select`/`Selector` | `flutter run --profile` + DevTools Performance & rebuild counts |
| NFR2 | Offline-first reliability | 100% of core features (FR1–FR9, FR11 buff usage) operate with zero errors in airplane mode; multi-table SQLite writes are atomic (`db.transaction`) | Airplane-mode walkthrough & force-close/reopen test |
| NFR3 | Security | Bearer token stored only in `flutter_secure_storage`; no hardcoded secrets (`--dart-define` for `API_URL` & `NEWS_API_KEY`); server derives user ownership strictly from token; release build obfuscated (`--obfuscate`); minimal Android permissions (`INTERNET`, `POST_NOTIFICATIONS`, `RECEIVE_BOOT_COMPLETED`) | `git grep` secret scan, API auth test, manifest check |
| NFR4 | Interaction & accessibility | Material 3 (`DESIGN.md`); WCAG contrast (≥ 4.5:1 normal text, ≥ 3:1 large/UI; no white on Sage, no Gold on light surface); Light & Dark mode; `ResponsiveBody` max-width 600 dp; touch targets ≥ 48×48 dp; Indonesian UI copy | Light/dark check on 360×640 phone, large phone, and tablet + larger text scale |
| NFR5 | Maintainability | Layered P12 architecture (`UI → GameStore → Services`); `flutter analyze` reports 0 errors/warnings; widgets ≤ ~150 lines; `const` constructors used where possible; feature-per-branch Git workflow | `flutter analyze` & PR code review |
| NFR6 | Compatibility & permissions | Android primary (emulator + physical device); runtime notification permission requested only on toggle; app remains functional if permission denied | Real-device test (Android 13+) |
| NFR7 | Data integrity | Integer Rupiah arithmetic; `GameRules` single source of math; SQLite `CHECK` constraints enforce `expense` has category and `saving` has `NULL`; restore runs in one atomic transaction | `flutter test` (`game_rules_test.dart`, `game_store_test.dart`) |
| NFR8 | Localization | All money formatted via `formatRupiah()` (`Rp 50.000`, `id_ID`); dates via `id_ID` locale with `initializeDateFormatting('id_ID')` | Visual check across Arena, History, Quest, Profile |
| NFR9 | Release readiness | Permanent `applicationId` (`id.ac.telkomuniversity.finquest`), version bump (`1.0.0+1`), signed release APK/AAB with `--split-debug-info`, complete `README.md` | Signed release build tested on physical phone |

## Verification Commands

```bash
# Static analysis & automated tests
flutter analyze
flutter test

# Run against local laptop server (Android emulator)
flutter run \
  --dart-define=API_URL=http://10.0.2.2:8000 \
  --dart-define=NEWS_API_KEY=YOUR_KEY

# Performance check (never benchmark in debug mode)
flutter run --profile \
  --dart-define=API_URL=http://10.0.2.2:8000 \
  --dart-define=NEWS_API_KEY=YOUR_KEY

# Signed & obfuscated release build
flutter build appbundle --release --obfuscate --split-debug-info=build/symbols \
  --dart-define=API_URL=https://your-api-or-lan-ip \
  --dart-define=NEWS_API_KEY=YOUR_KEY
```

## Notes

- **NewsAPI key caveat:** Passing `NEWS_API_KEY` via `--dart-define` keeps it out of Git, though compiled values remain extractable from an APK; document this as a known limitation in `README.md` for the P14 security review.
- **Local HTTP exception:** During campus demo with a laptop MySQL server (`http://10.0.2.2:8000` or LAN IP), Android cleartext traffic may be needed in debug/local builds; production/HTTPS servers require no cleartext traffic.
- **Guest uninstall risk:** Uninstalling the app deletes `finquest.db`. Users who want cross-device persistence must log in and run a manual backup (FR13).