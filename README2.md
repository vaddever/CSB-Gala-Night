# Staff Night 2026 — Event App

One Firebase project powers five pages. Everyone's identity (Staff ID, name,
contact) is captured once at the door and reused across the lucky draw,
karaoke, and quiz.

## The five pages

| File | Who uses it | What it's for |
|---|---|---|
| `register.html` | Every attendee, on their own phone | Scans the entrance QR → fills in Staff ID, name, contact → gets their pass + can opt into karaoke (picks gender) and the quiz |
| `admin.html` | You (the organizer) | Passcode-locked control room: see attendees, run the lucky draw, manage karaoke songs/pairing/voting, run the quiz |
| `display.html` | The projector / big screen | Switches automatically between idle, lucky draw, karaoke, and quiz views as you control them from `admin.html` |
| `karaoke-vote.html` | Attendees, on their own phone | Shows up automatically when a pair is performing, so the audience can score Performance & Singing |
| `quiz-play.html` | Attendees who joined the quiz | Shows the live question + countdown the moment you push it from `admin.html` |

Nobody types a URL by hand except you. `register.html` is the only link you
need to put on a poster/table QR; everything else is reached automatically
or via a button.

## 1. Create the Firebase project (10 minutes)

1. Go to [console.firebase.google.com](https://console.firebase.google.com) → **Add project** → give it a name → you can skip Google Analytics.
2. **Build → Firestore Database → Create database** → start in **production mode** → pick a region close to the Maldives (e.g. `asia-south1`).
3. **Build → Authentication → Sign-in method → Anonymous → Enable.** This is required — every page silently signs the visitor in anonymously so Firestore rules can tell "someone with the app open" apart from a random bot on the internet. No login screen is ever shown to attendees.
4. **Project settings (gear icon) → General → Your apps → Add app → Web (`</>`)** → register it (no hosting setup needed) → copy the `firebaseConfig` object it gives you.
5. Open `firebase-config.js` in this folder and paste your values in:

```js
window.FIREBASE_CONFIG = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  storageBucket: "...",
  messagingSenderId: "...",
  appId: "..."
};
window.ADMIN_PASSCODE = "pick-something-only-staff-know";
window.EVENT_NAME = "Staff Night 2026";
```

## 2. Set Firestore rules

In **Firestore Database → Rules**, paste this and click **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function signedIn() { return request.auth != null; }
    match /{document=**} {
      allow read, write: if signedIn();
    }
  }
}
```

**Honest note on security:** every page (including `admin.html`) authenticates
the same way — anonymously. The passcode on `admin.html` is a deterrent that
keeps casual attendees out, not a real access boundary; anyone determined
enough could read the page's source and find it. For a one-night internal
event this is a normal, accepted trade-off (it's exactly the same model as a
shared Wi-Fi password). If you ever reuse this for something with sensitive
data, that's the part to harden first — happy to help set up real
role-based access with Firebase custom claims if that day comes.

## 3. Deploy

Same pattern as your other projects: push this folder to a GitHub repo and
turn on **Pages** (Settings → Pages → deploy from `main` branch). Whatever
your `register.html` URL ends up being is the one link to put on the
entrance QR poster.

## 4. Run-of-night checklist

1. Open `admin.html` on your laptop, enter the passcode.
2. Click **Entrance QR** → display/print it at the door. Attendees scan it,
   fill the form once, and immediately get checked in.
3. Open `display.html` on the projector machine, full-screen it.
4. **Lucky draw:** pick time-based or manual stop, set duration, hit
   **Start lucky draw**. The big screen shuffles names; time-based stops
   itself, or hit **Stop now & reveal** any time. "Exclude past winners"
   keeps it fair across multiple draws in one night.
5. **Karaoke:** add your song list under Song bank ahead of time. Once
   enough people have opted in (gender selection happens at registration),
   hit **Auto-generate pairs** — it randomly matches one man with one woman
   and hands each pair the next unused song. **Put on stage** opens voting
   on every attendee's phone automatically (except the two performing).
   **Close voting** locks in their score. After all pairs are done,
   **Announce winning pair** reveals the leaderboard with the highest
   combined Performance + Singing score.
6. **Quiz:** add questions ahead of time (text + 4 options + mark the
   correct one). **Go to lobby** primes the big screen, **Next question**
   pushes the same question to everyone's phone at once with a countdown.
   Scoring rewards correctness first and speed second, so the fastest
   correct answer each round is what's announced — exactly matching how the
   final leaderboard is built. **Finish & show results** reveals the
   overall winner.
7. **Export CSV** any time from the Attendees tab if you need a list for
   HR/records afterward.

## Data model (for reference / debugging in the Firestore console)

- `attendees/{id}` — staffId, name, contact, checkedIn, wonLuckyDraw
- `karaoke_participants/{attendeeId}` — gender, paired, pairId
- `karaoke_songs/{id}` — title, assigned
- `karaoke_pairs/{id}` — maleName, femaleName, song, status, avgPerformance, avgSinging, totalScore
- `karaoke_votes/{pairId_voterId}` — performance, singing
- `quiz_questions/{id}` — text, options[4], correctIndex, order
- `quiz_participants/{attendeeId}` — totalScore
- `quiz_answers/{questionId_attendeeId}` — selectedIndex, clientTimeTakenMs
- `lucky_draw_winners/{id}` — history of every winner, in case you run multiple draws
- `state/display`, `state/lucky_draw`, `state/karaoke`, `state/quiz` — the live "what's happening right now" docs that drive the big screen

## A couple of things worth knowing

- **One device per attendee, ideally.** Your pass lives in that browser's
  local storage. If someone clears their browser or switches phones
  mid-event, re-entering the *same Staff ID* on `register.html` resumes
  their existing pass rather than creating a duplicate (important so nobody
  accidentally gets two lucky-draw entries).
- **The personal QR shown on the pass** isn't scanned by anything yet — it's
  there as a visual badge/proof of check-in. If you'd like a staff-operated
  scanning station at the door (e.g. for stricter pre-registration +
  day-of verification), that's a clean add-on — just say the word.
- **Everything needs internet** on every device, including the projector
  laptop. Test the venue Wi-Fi/data beforehand.
- Run through all three modules once end-to-end *before* the event with a
  couple of test phones — it'll catch any Firebase config typos early.
