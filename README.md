# Attendo — Facial Recognition Attendance

Attendo is a single-page web app that takes attendance using face recognition: students register their face once, then a live camera feed automatically recognizes and marks them present. It's built with **no build step at all** — the entire app is one `index.html` file using React via CDN script tags, deployed as a static site on Firebase Hosting.

## How it works

1. **Register** — a student's name, roll number, and class are entered, their face is captured via webcam, and `face-api.js` extracts a 128-point face descriptor, which is stored (along with their details) in the `students` Firestore collection.
2. **Verify** — a live camera feed continuously runs face detection every ~700ms. Each detected face's descriptor is compared against all registered students using `faceapi.FaceMatcher`; on a confident match, attendance is automatically logged to the `attendance` collection (de-duplicated per student per day). A "Manual Verify" button is also available for a one-off check.
3. **Dashboard** — shows today's present count vs. total registered students, plus a live list of today's check-ins.
4. **Report** — generate a present/absent list for any class and date by cross-referencing `students` against `attendance`.
5. **Danger Zone** — a confirmation-gated action to wipe all student and attendance records.

## Tech stack

- **React** (loaded via `<script>` tag, no JSX/bundler — the entire UI is built with `React.createElement`)
- **face-api.js** — in-browser face detection, landmarks, and recognition (TinyFaceDetector, FaceLandmark68Net, FaceRecognitionNet models)
- **Firebase Firestore** (compat SDK) — storage for student records (with face descriptors) and attendance logs
- **Firebase Hosting** — static site hosting
- Plain inline CSS-in-JS style objects (no Tailwind/CSS framework actually wired up at runtime — see note below)

## Project structure

```
.
├── public/
│   ├── index.html          # The entire application (markup, styles, logic)
│   ├── 404.html              # Default Firebase Hosting 404 page
│   ├── lib/                   # Vendored client libraries (served locally, no CDN dependency)
│   │   ├── react.js
│   │   ├── react-dom.js
│   │   ├── face-api.min.js
│   │   ├── firebase-app-compat.js
│   │   ├── firebase-firestore-compat.js
│   │   ├── babel.min.js          # Vendored but not referenced by index.html
│   │   └── tailwind-browser.js     # Vendored but not referenced by index.html
│   └── models/                 # face-api.js model weights (tiny face detector, landmarks, recognition)
├── firebase.json            # Firebase Hosting config (serves /public)
└── .firebaserc                # Points at the Firebase project "attendo-app-e2f10"
```

> `babel.min.js` and `tailwind-browser.js` are vendored in `public/lib/` but aren't actually loaded by `index.html` — likely leftovers from an earlier version of the app. They can be removed without affecting the current build.

## Data model (Firestore)

| Collection | Fields | Notes |
|---|---|---|
| `students` | `name`, `rollNumber`, `class`, `descriptor`, `registeredAt` | `descriptor` is the 128-float face embedding from face-api.js |
| `attendance` | `rollNumber`, `name`, `class`, `date`, `timestamp` | One doc per student per day; `date` is a `YYYY-MM-DD` string |

## Getting started

### Prerequisites

- A Firebase project with **Firestore** enabled
- The [Firebase CLI](https://firebase.google.com/docs/cli) (`npm install -g firebase-tools`) if you want to deploy or use `firebase serve`
- A browser with webcam access (camera permissions require HTTPS or `localhost`)

### Configuration

The Firebase config is hardcoded directly inside `public/index.html`:

```js
firebase.initializeApp({
  apiKey: "...",
  authDomain: "attendo-app-e2f10.firebaseapp.com",
  projectId: "attendo-app-e2f10",
  storageBucket: "attendo-app-e2f10.appspot.com",
  messagingSenderId: "...",
  appId: "..."
});
```

To point this at your own Firebase project, replace these values and update `.firebaserc` accordingly.

**Important:** this repo doesn't include any Firestore security rules. Since the app reads/writes Firestore directly from the client with no authentication, you should add rules restricting access (e.g. via Firebase Console or a `firestore.rules` file) before exposing this publicly — otherwise anyone with your Firebase config can read or write student data.

### Running locally

Since there's no build step, you just need to serve the `public/` folder over HTTP(S):

```bash
git clone https://github.com/Rajesh-041/Attendo.git
cd Attendo
firebase serve
```

Or with any static file server, e.g.:

```bash
npx serve public
```

Then open the printed local URL in your browser. Camera access will only work over `https://` or `http://localhost`.

### Deploying

```bash
firebase deploy
```

This publishes the contents of `public/` to Firebase Hosting under the project configured in `.firebaserc`.

## License

No license file is currently included in this repository.
