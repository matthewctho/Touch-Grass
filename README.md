# Touch Grass 🌱

**A gamified challenge-pack app that gets people off their phones and out into the real world.**
Built in two days at **SYNCS Hack 2026** (August 2026).

Users join themed *challenge packs*, complete real-world tasks (photo challenges, meeting new people), earn points, and redeem them for rewards. Photo submissions are checked automatically by Google's Gemini vision model. Social tasks need a mutual QR handshake, so nobody can earn points from another person without that person's consent.

<!-- Add screenshots: save them in docs/screenshots/ and uncomment
<p align="center">
  <img src="docs/screenshots/home.png" width="230">
  <img src="docs/screenshots/tasks.png" width="230">
  <img src="docs/screenshots/connect.png" width="230">
</p>
-->

## Features

- **Challenge packs and tasks**: browse packs, start one, and track per-pack progress with a progress bar
- **AI photo verification**: photos are uploaded to Firebase Storage and checked against the task's requirements by Gemini's multimodal API
- **Mutual QR connect**: one user shows a QR code and the other scans it. Points are awarded only once *both* sides have confirmed
- **Points ledger and rewards**: a running points balance with transactional reward redemption
- **Accounts**: Firebase Auth sign-up/login with user profiles in Firestore

## Tech stack

| Layer | Tech |
|---|---|
| App | Java, Android SDK (minSdk 30, targetSdk 37), Material Components |
| Backend | Firebase Auth, Cloud Firestore, Firebase Storage |
| AI | Gemini API (multimodal image + text) |
| QR | ZXing (`zxing-android-embedded`) |

## Architecture

```
app/src/main/java/com/mumble_jumble/touchgrass/
├── data/          # Backend / data layer — all Firebase + Gemini access goes through here
│   ├── AuthService.java
│   ├── FirestoreService.java
│   ├── StorageService.java
│   └── GeminiVerificationService.java
├── models/        # Firestore document models (User, ChallengePack, Task, Submission, Connection, ...)
├── activity/      # Pack list, task list, mutual connect, seed data
├── adapters/      # RecyclerView adapters
└── *.java         # Auth, home, progress, redeem screens
```

Activities never call Firebase directly. They go through the `data/` services, which take callback interfaces. That keeps the UI code separate from the backend.

### Firestore collections

| Collection | Purpose |
|---|---|
| `users` | Profile and the running `points` total |
| `challengePacks` / `tasks` | Pack and task definitions |
| `challengeProgress` | Per-user, per-pack progress. The doc ID is `{uid}_{packId}`, so progress is looked up directly instead of with a query |
| `submissions` | Photo submissions with a `status` of approved / rejected / pending |
| `connections` | QR handshake state between two users |

## Design decisions worth noting

**Redemption can't overspend.** `redeemReward` runs as a Firestore **transaction** (read, check balance, write) instead of a blind `FieldValue.increment(-cost)`. Two quick taps on "Redeem" can't both succeed off a stale balance, and the balance can never go negative.

**AI failure never silently rejects a user.** If the Gemini call fails or times out, the submission is set to `pending` for manual review instead of being rejected or faked as approved. Gemini's reply is requested as strict JSON (`{"approved": bool, "reason": "..."}`) and parsed defensively. Markdown code fences are stripped, and any missing field defaults to *not approved*.

**Consent is enforced in the data model.** A connection is created with only the initiator confirmed. The scanner's confirmation is the only thing that sets `bothConfirmed`. Per-side `pointsAwardedA/B` flags make sure a listener that fires again can never award points twice.

**No secrets in source.** The Gemini key is read from the gitignored `local.properties` into `BuildConfig` at build time.

## Running it locally

1. Clone the repo and open it in Android Studio.
2. **Firebase**: create a Firebase project with Auth (email/password), Firestore and Storage turned on. Download your own `google-services.json` into `app/`, replacing the existing one.
3. **Gemini**: get an API key from [Google AI Studio](https://aistudio.google.com/) and add it to `local.properties`:
   ```properties
   GEMINI_API_KEY=your_key_here
   ```
4. Build and run on a device or emulator (API 30+). A camera is needed for photo tasks and QR scanning.
5. Open `SeedDataActivity` once to fill Firestore with sample challenge packs and tasks.

## Team and my role

Built by a team of four at SYNCS Hack 2026:

- **Matthew Ho**
- **Nikhil Subramani**
- **Sanjeev Ragothaman**
- **Manunandhan Stalinkumar**
