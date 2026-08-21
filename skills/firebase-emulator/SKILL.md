---
name: firebase-emulator
description: Set up a real local test environment for a Firebase/Firestore project using the Firebase Emulator Suite. USE THIS SKILL PROACTIVELY, without being asked, whenever a project uses Firestore or Firebase Auth and there is no working local environment to test against — and ALWAYS when the project's CLAUDE.md has no "Test environment" section, when an agent needs a base URL or test credentials and cannot find them, when a test would otherwise have to run against a deployed or shared Firebase project, or when someone is about to point tests, seeds or scripts at real data. Covers firebase.json emulator config, connecting the app via FIRESTORE_EMULATOR_HOST / FIREBASE_AUTH_EMULATOR_HOST, seeding and exporting fixtures, anonymised realistic data, and writing the "Test environment" section that every other agent depends on.
---

# Firebase emulator: a local environment worth testing against

The goal is not "an emulator is installed". The goal is: anyone — a person or an
agent — can run one command, get a working app with known users and known data,
break it, and reset it. Until that is true and written down, testing is guesswork
and the only alternative is pointing at something real.

Work through the six steps below in order. Do not skip step 6.

---

## 1. Add the `emulators` block to `firebase.json`

Fixed ports, so that URLs are stable and can be written down. `singleProjectMode`
so a mismatched project ID fails loudly instead of silently splitting data across
two namespaces. Reference the real rules and index files, so the emulator enforces
the same rules the deployment will.

```json
{
  "firestore": {
    "rules": "firestore.rules",
    "indexes": "firestore.indexes.json"
  },
  "emulators": {
    "auth": { "port": 9099 },
    "firestore": { "port": 8080 },
    "ui": { "enabled": true, "port": 4000 },
    "singleProjectMode": true
  }
}
```

If `firestore.rules` or `firestore.indexes.json` do not exist yet, create them —
an emulator running in open mode teaches you nothing about whether your rules work.
A minimal starting `firestore.rules` should still deny by default and grant
explicitly.

Start it with:

```
firebase emulators:start --project demo-<projectname>
```

A project ID beginning with `demo-` has no cloud counterpart at all: the SDKs
refuse to fall back to a real backend. Prefer it.

---

## 2. Connect the app to the emulators

Two mechanisms; use both, because they cover different halves of the stack.

**Server side / Admin SDK / scripts** — environment variables. The Admin SDK reads
them automatically and never touches the cloud when they are set:

```
FIRESTORE_EMULATOR_HOST=127.0.0.1:8080
FIREBASE_AUTH_EMULATOR_HOST=127.0.0.1:9099
```

Put them in the dev-only env file the app already loads (`.env.local`,
`.env.development`) — never in the production env file, never committed alongside
real credentials.

**Client side / Web SDK** — explicit calls, immediately after creating the
instances and before any read or write:

```js
import { getFirestore, connectFirestoreEmulator } from 'firebase/firestore';
import { getAuth, connectAuthEmulator } from 'firebase/auth';

const db = getFirestore(app);
const auth = getAuth(app);

if (import.meta.env.DEV) {
  connectFirestoreEmulator(db, '127.0.0.1', 8080);
  connectAuthEmulator(auth, 'http://127.0.0.1:9099', { disableWarnings: true });
}
```

Guard it on an explicit dev/test flag, never on `NODE_ENV !== 'production'` alone —
that is how a preview build ends up talking to a laptop that is not there.

Verify the wiring before moving on: write one document from the app and confirm it
appears in the Emulator UI at `http://127.0.0.1:4000`. If it does not, the app is
still talking to the cloud and everything after this point is fiction.

---

## 3. Seed data, and commit the seed

Write a seed script that runs **against the emulators** (the env vars from step 2
must be set in its process) and creates:

- `qa@test.local` / `test1234` — the default user, the one every example uses
- one user per role the app actually distinguishes: `admin@test.local`,
  `editor@test.local`, `viewer@test.local`, all with `test1234`
- the minimum documents each role needs to reach a meaningful screen — not an
  empty account that only ever shows the empty state

Use the Admin SDK so you can create users with fixed UIDs and set custom claims:

```js
// seed.mjs — requires FIRESTORE_EMULATOR_HOST and FIREBASE_AUTH_EMULATOR_HOST
import { initializeApp } from 'firebase-admin/app';
import { getAuth } from 'firebase-admin/auth';
import { getFirestore } from 'firebase-admin/firestore';

initializeApp({ projectId: 'demo-app' });

const users = [
  { uid: 'u-qa',     email: 'qa@test.local',     role: 'user'   },
  { uid: 'u-admin',  email: 'admin@test.local',  role: 'admin'  },
  { uid: 'u-editor', email: 'editor@test.local', role: 'editor' },
  { uid: 'u-viewer', email: 'viewer@test.local', role: 'viewer' },
];

for (const u of users) {
  await getAuth().createUser({ uid: u.uid, email: u.email, password: 'test1234' });
  await getAuth().setCustomUserClaims(u.uid, { role: u.role });
  await getFirestore().doc(`users/${u.uid}`).set({ email: u.email, role: u.role });
}
```

Then freeze the result so nobody has to re-run the script to get a known state:

```
firebase emulators:export ./seed --force
```

Commit `./seed`. From then on, the documented start command loads it and writes
changes back on exit:

```
firebase emulators:start --project demo-app --import ./seed --export-on-exit ./seed
```

Two useful variants worth documenting: `--import ./seed` without
`--export-on-exit` gives a throwaway session that resets on every restart — often
what you want while testing. Keep `--export-on-exit` for when you are deliberately
updating the fixture, and review that diff like any other code change.

---

## 4. If you need realistic data

Sometimes the shape of real data is the thing that breaks the app: long names,
missing optional fields, records with 400 children, unicode, empty strings that
were supposed to be nulls. A hand-written seed will never contain those.

The way to get it:

1. On a **trusted machine** that already has legitimate access to real data — a
   maintainer's laptop, not CI, not an agent — export a snapshot.
2. Run an **anonymisation script** over that snapshot before it goes anywhere:
   replace names, emails, phone numbers, addresses, free-text notes, avatars and
   any external IDs; keep lengths, cardinalities, distributions, timestamps and
   the relational structure, because those are the parts that expose bugs.
3. Import the anonymised result into the emulator and export it as a fixture with
   `firebase emulators:export`.
4. Review the fixture by hand before committing. Grep it for `@` and for known
   real domains, surnames and customer names. Anonymisation scripts miss fields —
   especially free text and nested objects.

**Never point the app, a test, an agent or a seed script at production.** Not
read-only, not "just to check", not with a filter. The emulator's whole value is
that a mistake inside it costs nothing; a connection to production discards that
in one line. If real data is needed and cannot be safely anonymised, work with a
smaller synthetic dataset and say so.

---

## 5. Write the "Test environment" section in the project's `CLAUDE.md`

This is the deliverable other agents actually consume. `webtester` refuses to run
without it. Use exactly this heading, and fill in the real values:

```markdown
## Test environment

Local only. Never test against a deployed environment.

**Start**

    firebase emulators:start --project demo-app --import ./seed
    npm run dev

**URLs**

- App: http://127.0.0.1:5173
- Emulator UI: http://127.0.0.1:4000
- Firestore emulator: 127.0.0.1:8080
- Auth emulator: 127.0.0.1:9099

**Test users** (all password `test1234`)

| Email              | Role   | Use for                       |
| ------------------ | ------ | ----------------------------- |
| qa@test.local      | user   | default happy path            |
| admin@test.local   | admin  | settings, user management     |
| editor@test.local  | editor | create and edit content       |
| viewer@test.local  | viewer | read-only / permission denial |

**Reset**

Stop the emulators and restart without `--export-on-exit`; the committed `./seed`
fixture is reloaded and any local changes are discarded.

    firebase emulators:start --project demo-app --import ./seed
```

Keep it short and literal. Commands that can be pasted, not described.

---

## 6. Prove it starts

Do not report this done from having written the files. Actually:

1. Run the start command from the section you just wrote, in a clean shell.
2. Confirm the emulator UI answers on its port and shows the imported collections.
3. Log into the app as `qa@test.local` / `test1234` through the real UI.
4. Confirm one role restriction behaves as expected — `viewer@test.local` is
   denied something `editor@test.local` can do. That proves the rules file is
   actually loaded, which is the single most common silent failure here.
5. Stop and restart, and confirm you get the same state back.

If any step fails, fix it now. A test environment that only works on the machine
where it was written is worse than none, because it will be documented as working.

---

## What the emulator will not tell you

The emulator is a good local Firestore. It is not your deployment, and these
differences are the ones that reach users:

- **Composite indexes.** The emulator will happily serve queries that production
  rejects with `FAILED_PRECONDITION: The query requires an index`. Every
  multi-field or range+order query must be present in `firestore.indexes.json`
  and deployed. Verify against the real project, not locally.
- **Volume.** The emulator holds your seed in memory. It says nothing about
  collections with millions of documents, pagination that degrades, fan-out
  writes hitting per-document write limits, or hot-spotted sequential keys.
- **Latency and consistency.** Everything is local and instant. Race conditions,
  optimistic-UI flicker, retry behaviour, offline persistence and slow-network
  states are invisible here — test those against a real deployment, and with
  throttling in the browser.
- **Everything in front of the app.** CDN and cache headers, hosting rewrites,
  auth domains and third-party cookie behaviour, CORS, App Check, rate limits,
  Cloud Functions cold starts, real quotas and billing limits. None of it exists
  locally.

Use the emulator to be sure the logic and the rules are right. Confirm indexes,
scale and infrastructure against the actual deployment before trusting them.
