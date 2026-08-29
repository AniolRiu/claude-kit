---
name: firestore-rules
description: Write Firestore and Storage security rules and prove them against the emulator, testing what must be denied as carefully as what must be allowed. Use when adding a collection or a field, when changing who may read or write something, when a feature relies on the client hiding a control, when rules exist but have never been tested, or before shipping anything that touches other people's data. Security lives in the rules, not in the client — a hidden button is not an authorization check. Covers default-deny structure, validating the shape of a write, the negative cases a suite must contain, and what rules cannot enforce at all.
---

# Firestore and Storage rules, and proving them

The client is not a security boundary. Anyone can call the SDK directly with their own
token and any payload they like, so hiding a button, disabling a field or checking a role
in a component decides what people *see*, never what they *can do*. The rules are the
only thing standing between a user and the database.

Which means untested rules are not security, they are an intention. This skill is mostly
about the testing.

## 1. Structure: deny by default

Start from nothing allowed and open exactly what each collection needs. A rule that ends
in a broad `allow read, write: if request.auth != null` grants every signed-in user the
whole database.

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function signedIn()      { return request.auth != null; }
    function isSelf(uid)     { return signedIn() && request.auth.uid == uid; }
    function role()          { return request.auth.token.role; }

    match /users/{uid} {
      allow read:   if isSelf(uid);
      allow create: if isSelf(uid) && validUser();
      allow update: if isSelf(uid) && validUser()
                    && request.resource.data.role == resource.data.role;
      allow delete: if false;
    }
  }
}
```

Three things that matter more than they look:

- **Never trust a field the client sends about itself.** A user who can write their own
  `role`, `ownerId`, `plan` or `verified` field owns the system. Either the field is
  immutable in the rule, as above, or it lives in a custom claim, or it is written only
  by a server.
- **Validate the shape on write**, not just the identity: which keys are allowed, their
  types, and that nothing else comes through. `request.resource.data.keys().hasOnly([...])`
  is the difference between a schema and a hope.
- **Read rules are not filters.** A rule cannot narrow what a query returns; it rejects a
  query it cannot prove is allowed. Rules and queries are designed together.

## 2. Test against the emulator, with the real rules file

Use `@firebase/rules-unit-testing` against the running emulator, loading the same
`firestore.rules` the deployment uses. If the emulator is not set up yet, that comes
first — see the `firebase-emulator` skill.

```js
import { initializeTestEnvironment, assertSucceeds, assertFails }
  from '@firebase/rules-unit-testing';

const env = await initializeTestEnvironment({
  projectId: 'demo-rules',
  firestore: { rules: readFileSync('firestore.rules', 'utf8') },
});

const alice = env.authenticatedContext('alice').firestore();
const bob   = env.authenticatedContext('bob').firestore();
const anon  = env.unauthenticatedContext().firestore();
```

Seed fixtures through `env.withSecurityRulesDisabled(...)` — setting up a document should
not depend on the rule you are about to test.

## 3. The negative cases are the test

A suite that only checks what should work proves nothing at all: rules that allow
everything pass it perfectly. For every rule, the case that earns its place is the one
that must **fail**.

Cover, for each collection:

- **Unauthenticated.** `assertFails(anon.doc(...).get())`.
- **Authenticated but somebody else.** Bob reading, writing and deleting Alice's document.
- **The wrong role.** A viewer doing what only an editor may.
- **Field tampering.** The owner writing their own `role`, `ownerId` or any field a server
  is supposed to control — this is the one people forget, and the expensive one.
- **Delete.** Almost always forgotten, and almost never intended.

Name the tests after the sentence they prove: `a viewer cannot publish`, not
`test update 3`.

## 4. Storage rules too

The same applies to `storage.rules`, plus two things Firestore does not have: **size and
content type**. An upload path open to any signed-in user is an open file host.

```
match /users/{uid}/{file} {
  allow write: if isSelf(uid)
               && request.resource.size < 5 * 1024 * 1024
               && request.resource.contentType.matches('image/.*');
}
```

## 5. What rules cannot do

Do not try to express these in rules; they need a server, a callable, or a scheduled job:

- Rate limiting, quotas, and anything about cost.
- Constraints across several documents, or aggregates ("no more than 10 per user").
- Anything needing a secret, an external service, or the current time beyond a comparison.
- Validation that requires reading many documents — every `get()` in a rule is billed and
  counts against a hard limit per evaluation.

## 6. Keep these tests

Unlike an exploratory browser script, rules tests belong in the repository and run in CI.
They are cheap, they are fast, and they are the only regression net for the one part of
the system where a mistake is not a bug but a breach.
