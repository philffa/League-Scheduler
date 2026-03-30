# FFA Scheduler — Firebase Setup Guide

## Files in this package

| File | Purpose |
|---|---|
| `index.html` | Public landing page + exec login |
| `dashboard.html` | Live schedule dashboard (auth-protected) |
| `ffa-scheduler.html` | Full schedule editor (save button syncs to Firestore) |

---

## Step 1 — Create a Firebase project

1. Go to https://console.firebase.google.com
2. Click **Add project** → name it (e.g. `ffa-scheduler`)
3. Disable Google Analytics (not needed) → **Create project**

---

## Step 2 — Enable Authentication

1. In Firebase Console → **Build → Authentication**
2. Click **Get started**
3. Under **Sign-in method**, enable:
   - **Email/Password** ✓
   - **Google** ✓ (optional, for Google sign-in)
4. Under **Users** tab, click **Add user** → add each exec committee member's email + password

---

## Step 3 — Create Firestore database

1. In Firebase Console → **Build → Firestore Database**
2. Click **Create database**
3. Choose **Start in production mode** → select your region (e.g. `australia-southeast1`) → **Enable**
4. Go to **Rules** tab and replace the default rules with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Only authenticated users can read/write the schedule
    match /ffa/schedule {
      allow read, write: if request.auth != null;
    }
  }
}
```

5. Click **Publish**

---

## Step 4 — Get your Firebase config

1. In Firebase Console → **Project Settings** (gear icon top-left)
2. Under **Your apps** → click **</>** (Web) → Register app → name it `ffa-web`
3. Copy the `firebaseConfig` object shown

It looks like this:
```js
const firebaseConfig = {
  apiKey: "AIzaSy...",
  authDomain: "ffa-scheduler.firebaseapp.com",
  projectId: "ffa-scheduler",
  storageBucket: "ffa-scheduler.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123"
};
```

---

## Step 5 — Paste config into all three files

In **`index.html`**, **`dashboard.html`**, and **`ffa-scheduler.html`**, find the comment:

```js
// ── PASTE YOUR FIREBASE CONFIG HERE ──
const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  ...
```

Replace the placeholder object with your real config. Do this in all three files.

---

## Step 6 — Add Firestore sync to ffa-scheduler.html

In `ffa-scheduler.html`, after the closing `</script>` tag of the main script block, add:

```html
<script type="module">
  import { initializeApp } from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js';
  import { getFirestore, doc, setDoc }
    from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-firestore.js';
  import { getAuth, onAuthStateChanged }
    from 'https://www.gstatic.com/firebasejs/10.12.0/firebase-auth.js';

  const firebaseConfig = { /* YOUR CONFIG */ };
  const app = initializeApp(firebaseConfig);
  const db  = getFirestore(app);
  const auth = getAuth(app);

  // Save state to Firestore
  window.saveToFirestore = async function() {
    const user = auth.currentUser;
    if (!user) { alert('Sign in first to sync to Firestore.'); return; }
    try {
      await setDoc(doc(db, 'ffa', 'schedule'), window._ffaState || state);
      toast('Synced to Firestore ✓', 'success');
    } catch(e) {
      toast('Sync failed: ' + e.message, 'error');
    }
  };
</script>
```

Then add a **Sync** button near the Generate button in `ffa-scheduler.html`:
```html
<button class="btn btn-green btn-full" onclick="saveToFirestore()">☁ Sync to Firebase</button>
```

---

## Step 7 — Deploy to GitHub Pages

1. Put all three files in your GitHub Pages repo (same folder as your other FFA tools)
2. Commit and push
3. Your URLs will be:
   - `https://yourusername.github.io/yourrepo/index.html` — landing page
   - `https://yourusername.github.io/yourrepo/dashboard.html` — exec dashboard
   - `https://yourusername.github.io/yourrepo/ffa-scheduler.html` — scheduler (existing)

---

## How the sync works

```
ffa-scheduler.html  ──[Save to Firestore]──▶  Firestore /ffa/schedule
                                                        │
                              ┌─────────────────────────┘
                              ▼
                    dashboard.html (all execs)
                    onSnapshot() ← real-time listener
                    Updates instantly for every logged-in user
```

Every exec with a login will see the schedule update live the moment you click **Sync to Firebase** in the scheduler — no page refresh needed.

---

## Adding / removing exec members

Go to **Firebase Console → Authentication → Users** and add or delete users there. No code changes needed.
