# trustbank

# TrustBank

A security-focused banking app for Android, built with Kotlin and Jetpack Compose, backed by a custom REST API. Every account view is gated behind biometric authentication, and the local database is encrypted at rest — this is the third and most advanced project in the portfolio, pairing an Android client with its own server.

This is the third project in a three-part Android portfolio (EchoNotes → SnapSpend → TrustBank). It builds on SnapSpend's local persistence by adding encryption, biometric access control, and a real backend the app syncs with — see the companion [TrustBankServer](../TrustBankServer/README.md) repo for the API.

## Features

- **Biometric lock screen** — the app opens to a lock screen and won't show any account data until fingerprint/face (or device credential) authentication succeeds, via Android's `BiometricPrompt`.
- **Encrypted local database** — the Room database is encrypted at rest with SQLCipher, so the `.db` file on disk isn't readable without the passphrase.
- **Keystore-backed passphrase** — that passphrase is 32 random bytes generated once on first launch, then stored in `EncryptedSharedPreferences`, whose own encryption key lives in the Android Keystore (hardware-backed on most devices). The passphrase never appears anywhere as a fixed string in source.
- **Multiple accounts with running balances** — create checking/savings/etc. accounts; each one's balance is a running total updated on every transaction rather than recomputed by summing history each time.
- **Deposits, withdrawals, and transfers** — a transfer between two accounts is stored as two linked transaction rows (`TRANSFER_OUT` / `TRANSFER_IN`, joined by `linkedTransferId`), so it can be displayed or reasoned about as one logical transfer even though it's two ledger entries — the same double-entry pattern the server enforces.
- **Server-backed with offline fallback** — the app talks to the [TrustBankServer](../TrustBankServer/README.md) backend over Retrofit and treats it as the source of truth, caching every response into Room. If the server is unreachable, writes fall back to a local-only Room insert so the app keeps working, and a background reconciliation is left as a known simplification (no id-remapping or conflict resolution yet).

## Tech Stack

- **Kotlin**
- **Jetpack Compose** (Material 3) for the UI
- **Navigation Compose** with per-screen **ViewModels** (`AccountViewModel`, `TransactionViewModel`)
- **Room + SQLCipher** (`net.zetetic:android-database-sqlcipher`) for an encrypted local database — 2 entities: `Account`, `Transaction` (cascading foreign key from `Transaction.accountId` to `Account.id`)
- **AndroidX Security Crypto** (`EncryptedSharedPreferences` + `MasterKey`, Android Keystore-backed) for storing the database passphrase
- **AndroidX Biometric** (`BiometricPrompt`) for the lock screen
- **Retrofit + Gson + OkHttp** (with a logging interceptor) for talking to the backend
- Min SDK 26, Target SDK 37

## Project Structure

| File | Purpose |
|---|---|
| `MainActivity.kt` | App entry point and navigation host (a `FragmentActivity`, required to host `BiometricPrompt`) |
| `data/Account.kt` / `data/Transaction.kt` | The 2-table schema — accounts and their transactions, cascading on delete |
| `data/AppDatabase.kt` | Room database definition, opened through a SQLCipher `SupportFactory` instead of Room's default plain-SQLite engine |
| `data/AccountRepository.kt` / `data/TransactionRepository.kt` | Server-first, local-cache-fallback data layer — every write tries the API first, then Room |
| `security/SecurePassphraseProvider.kt` | Generates and stores the database's encryption passphrase via `EncryptedSharedPreferences` |
| `security/BiometricAuthHelper.kt` | Wraps `BiometricPrompt` — checks availability and runs the authentication flow |
| `network/TrustBankApiService.kt` / `network/ApiModels.kt` | Retrofit interface and request/response models for the backend |
| `network/RetrofitClient.kt` | Retrofit/OkHttp client configuration (points at `10.0.2.2:4000`, the Android emulator's alias for the host machine) |
| `ui/LockScreen.kt` | The biometric lock screen shown before any account data |
| `ui/AccountsOverviewScreen.kt` / `ui/AddAccountScreen.kt` / `ui/AccountDetailScreen.kt` | Account list, creation, and detail/transaction-history views |
| `ui/TransferScreen.kt` | Transfer flow between two accounts |
| `ui/theme/` | The app's Compose theme |

## Known Limitations

The offline fallback is a known simplification: if a write happens while the server is unreachable, it's saved locally with a Room-generated id but never retried against the server, so there's no real reconciliation or conflict resolution once connectivity returns. `RetrofitClient` is hardcoded to `10.0.2.2:4000`, the Android emulator's special alias for the host machine — running on a physical device requires pointing it at the host machine's actual LAN IP instead. As with the other two apps in this portfolio, Room's `fallbackToDestructiveMigration()` means a schema version bump wipes local data rather than migrating it.

## Getting Started

1. Clone this repo and the [TrustBankServer](../TrustBankServer/README.md) repo, and start the server first (`npm start` — see its README) so the app has something to talk to.
2. Open this project in Android Studio and let Gradle sync — it pulls in Room, SQLCipher, AndroidX Security Crypto, Biometric, Navigation Compose, and Retrofit.
3. Run on an emulator running Android 8.0 (API 26) or higher. The default backend URL (`10.0.2.2:4000`) only resolves correctly from an emulator on the same machine running the server.
4. The emulator needs a biometric enrolled (or a screen lock set as a device credential fallback) to get past the lock screen — on an AVD, enroll a fingerprint via Extended Controls → Fingerprint.

# TrustBankServer

A small local REST API backing the [TrustBank](../Trustbank/README.md) Android app, built with Node.js and Express. It mirrors the same accounts/transactions shape as the app's own encrypted Room database, and knows nothing about the app's encryption or biometric lock — that's entirely the phone's concern.

## Features

- **Account management** — create, list, fetch, and delete bank accounts, each with a running balance.
- **Deposits & withdrawals** — post a transaction against a single account; the account's balance is adjusted server-side in the same request.
- **Transfers between accounts** — a transfer writes two linked transaction rows (`TRANSFER_OUT` on the source account, `TRANSFER_IN` on the destination, cross-referenced by `linkedTransferId`) and adjusts both balances atomically within the request — the same double-entry ledger pattern the Android app mirrors locally.
- **Per-account and global transaction history** — fetch a single account's transactions in reverse-chronological order, or the most recent transactions across all accounts.

## Tech Stack

- **Node.js** with **Express**
- **`node:sqlite`** (Node's built-in synchronous SQLite module — `DatabaseSync`) for persistence, no external database dependency
- **cors**, for the Android app (running on a different origin/port) to call the API

## API Reference

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/accounts` | List all accounts |
| `POST` | `/accounts` | Create an account (`name`, `type`, optional `balance`) |
| `GET` | `/accounts/:id` | Fetch a single account |
| `DELETE` | `/accounts/:id` | Delete an account (its transactions cascade-delete with it) |
| `GET` | `/accounts/:id/transactions` | List an account's transactions, newest first |
| `GET` | `/transactions?limit=20` | List the most recent transactions across all accounts |
| `POST` | `/accounts/:id/transactions` | Post a `DEPOSIT` or `WITHDRAWAL` against an account |
| `POST` | `/transfer` | Transfer between two accounts (`fromAccountId`, `toAccountId`, `amount`) |

## Schema

Two tables, matching the Android app's Room schema:

- **`accounts`** — `id`, `name`, `type`, `balance`
- **`transactions`** — `id`, `accountId` (foreign key → `accounts.id`, cascades on delete), `type`, `amount`, `title`, `date`, `note`, `linkedTransferId` (set only on transfer rows, points at the sibling row on the other account)

## Getting Started

1. `npm install`
2. `npm start` — starts the server on `http://localhost:4000` (`0.0.0.0`, so it's also reachable from an Android emulator at `10.0.2.2:4000`).
3. The SQLite database file (`trustbank.db`) is created automatically on first run in the project directory — no separate database setup needed.
4. Run the [TrustBank](../Trustbank/README.md) Android app against it — its default configuration already points at `10.0.2.2:4000` for the emulator.

## Known Limitations

This is a local development backend with no authentication on the API itself — access control (the biometric lock, encrypted local storage) lives entirely on the Android app side, and this server trusts any client that can reach it. There's also no input validation beyond basic required-field checks, and no real migration story if the schema changes (`CREATE TABLE IF NOT EXISTS` only creates tables that don't yet exist; it won't alter existing ones).
