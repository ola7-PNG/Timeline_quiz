# A Century of Live Music — Art Installation Quiz

A two-screen interactive quiz for an art installation. Visitors scan a QR code to take a 20-question quiz on their phones; a projector displays the live leaderboard, updating in real time as people finish.

## What's in this repo

- **`index.html`** — the player's quiz (run on phones via QR code)
- **`projector.html`** — the live leaderboard view (run on the projector / display screen)

Both pages share a Firebase Firestore database, so scores submitted from phones appear on the projector within seconds.

## Setup checklist

### 1. Push to GitHub Pages

1. Create a GitHub repo (e.g. `concert-quiz`).
2. Upload `index.html`, `projector.html`, and `README.md`.
3. **Settings → Pages → Source:** "Deploy from a branch", **Branch:** `main` → `/ (root)` → **Save**.
4. After ~60 seconds, your two URLs will be:
   - Player: `https://<username>.github.io/<repo>/`
   - Projector: `https://<username>.github.io/<repo>/projector.html`

### 2. Lock down your Firestore Rules (do this before opening to the public)

Right now your database is in **test mode**, which means anyone in the world can read, write, edit, AND delete entries. For a public installation, that's a serious risk — someone could trash the leaderboard.

Go to your Firebase console at [console.firebase.google.com](https://console.firebase.google.com) → **timeline-quiz-62280** → **Firestore Database** → **Rules**, and replace the contents with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /leaderboard/{entry} {
      allow read: if true;
      allow create: if
        request.resource.data.keys().hasAll(['name', 'score', 'correct', 'ts'])
        && request.resource.data.name is string
        && request.resource.data.name.size() > 0
        && request.resource.data.name.size() <= 24
        && request.resource.data.score is int
        && request.resource.data.score >= 0
        && request.resource.data.score <= 5000
        && request.resource.data.correct is int
        && request.resource.data.correct >= 0
        && request.resource.data.correct <= 25
        && request.resource.data.ts is int;
      allow update, delete: if false;
    }
  }
}
```

Click **Publish**. These rules:
- Allow anyone to view scores (so the projector can read).
- Allow new score submissions, but only with valid fields and within sane limits (no oversized names, no impossible scores, no script injection).
- Block edits and deletions entirely — entries are append-only.

If you ever need to wipe the leaderboard manually (e.g. between events), do it from the Firebase console: Firestore Database → leaderboard collection → delete documents.

### 3. Generate the QR code

Generate a QR code that links to the player URL (`https://<username>.github.io/<repo>/`). Free QR code generators that work fine: [qr-code-generator.com](https://www.qr-code-generator.com), [qrcode-monkey.com](https://www.qrcode-monkey.com).

Print the QR code as part of your installation signage — make it large enough to scan from a few feet away.

### 4. Set up the projector

On the day of the installation:

1. Connect your projector to a laptop.
2. Open `https://<username>.github.io/<repo>/projector.html` in a browser.
3. Press **F11** (Windows/Linux) or **Ctrl+Cmd+F** (Mac) to enter full-screen.
4. Disable screen sleep / display sleep on the laptop:
   - **Mac:** System Settings → Lock Screen → "Turn display off when inactive: Never"
   - **Windows:** Settings → System → Power → Screen → Never

That's it — leave it running. New scores appear with a glowing animation and the leaderboard reorders automatically.

### 5. Test the flow

Before opening to the public, run an end-to-end test:

1. Open the player URL on your phone, take the quiz with a test name like "TEST_1".
2. Watch the projector — within 1-3 seconds you should see "TEST_1" appear with a glowing animation.
3. The connection indicator in the top-right of the projector should read "Live" with a pulsing red dot. If it says "Offline", something's wrong with Firebase access — check your config and rules.

After testing, delete the test entries from the Firestore console.

## How the projector view works

- **Real-time updates** — Uses Firestore's `onSnapshot` API, which pushes new scores to the projector the instant they're written. No polling, no refresh.
- **"Just-in" animation** — When a brand-new score arrives, that row glows lavender and slides in to draw attention.
- **Top 20 displayed** — The leaderboard shows the top 20 scores by default. Change `TOP_N` in `projector.html` if you want more or fewer.
- **Live stats bar** — Above the leaderboard: total players, top score, average score. Stats animate when they change.
- **Connection indicator** — Top-right corner: green pulsing dot when live, lavender when connecting, grey when offline.
- **Podium styling** — Top 3 spots get gold, silver, and bronze accent colours.
- **Pure CSS animations** — Everything (vinyl records, equalizer, stage lights, shimmer) runs without external libraries.

## Customisation

### Change the leaderboard size
In `projector.html`, find `const TOP_N = 20;` near the top of the script and change the number.

### Change the colour palette
Both files share the same CSS variables at the top of their `<style>` blocks: `--deep-purple`, `--lavender`, `--bg`, etc. Edit them in both files to keep the player and projector views consistent.

### Reset the leaderboard between events
Two options:
1. **Delete all docs** in the Firebase console (Firestore Database → leaderboard → bin icons on each doc).
2. **Use a different collection per event** — change `FIREBASE_COLLECTION` in `index.html` and `projector.html` to e.g. `leaderboard_2026_event_1`.

### Edit the questions
In `index.html`, find the `QUESTIONS` array. Each entry has `era`, `q`, `options`, `correct` (zero-indexed), and `fact`. Add, remove, or rewrite freely. If you change the question count, also update the "20" references in:
- The vinyl label (`★ 20 Tracks ★`)
- The ticket on the welcome screen (`20 questions`)
- The quiz counter (`Track 1 of 20`)
- The score-tier thresholds in `getTier()`
- The `correct/20` text in `projector.html`

## Troubleshooting

**"Connection indicator says Offline"** → Open the browser console (F12). If you see permission errors, your Firestore Rules may be blocking reads. Make sure `allow read: if true;` is published.

**"Scores aren't showing on the projector"** → Confirm both pages use the same Firebase config and the same `FIREBASE_COLLECTION` value. Take the quiz on a phone and check the Firestore console (`leaderboard` collection) — if entries appear there but not on the projector, it's a projector subscription issue (refresh the page).

**"Players see a network error"** → The phone may be on a slow Wi-Fi. Scores fall back to localStorage on the phone, so they're not lost — but they won't appear on the projector until the network recovers and the page is reloaded.

**"Someone wrote something inappropriate"** → Delete the entry directly in the Firestore console. To prevent it, you could add a profanity filter to `index.html` before the score is submitted.

## Costs

Firebase's free Spark tier covers:
- 50,000 reads/day
- 20,000 writes/day
- 1 GB storage

For an art installation with even a few hundred visitors, this is far more than enough. The projector subscription uses real-time updates rather than repeated polling, so reads scale with new score submissions, not with viewing time.

## Tech notes

- Two static HTML files. No build step, no dependencies, no server.
- Firebase Web SDK loaded on demand from Google's CDN.
- All animations are pure CSS or `requestAnimationFrame`.
- localStorage is written as a backup on every score submission, so a phone briefly losing connection doesn't lose the score.
- Mobile-responsive on the player side; the projector view is sized for 1080p+ landscape displays.

## Credits

- Brand palette inspired by Logoexposure (Spheris AI project).
- Concert history compiled from the timeline poster and verified against published sources.
