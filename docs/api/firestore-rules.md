# Firestore Security Rules — deployment & verification guide

Backend fix for issues **#3** / **#12**: the live Firestore rules deny every
unauthenticated request, so the Blog and Feedback wall are empty for all
visitors and visitor-submitted feedback is silently lost.

The rules themselves do not live in this repository — they run in Firebase
project `ippevent-6403b`. This PR adds `firestore.rules` to the repo as the
reviewable source of truth plus this guide, and aligns the client queries in
`index.html` with the rules (see "Why the client changed" below).

## What the rules do

| Collection | Read | Create | Update / Delete |
|---|---|---|---|
| `posts` | published docs: anyone; drafts: admin only | admin only | admin only |
| `feedback` | anyone | anyone (validated payload) | admin only |
| everything else | deny all | deny all | deny all |

The admin is the single account `nitin.goswami.office@gmail.com`
(`isAdmin()` in `firestore.rules`). The client-side `fbIsAdmin` check is
cosmetic; these rules are the security boundary.

## Deploy

Option A — Firebase CLI (recommended; repo is already wired via `firebase.json`):

```bash
npm install -g firebase-tools
firebase login
firebase deploy --only firestore:rules
```

Option B — console paste: Firebase console → Firestore → Rules → paste the
contents of `firestore.rules` → Publish.

## Why the client changed too

Firestore rules are **not filters**: a `list` request is only permitted if the
rules can prove *every* document it could return satisfies the condition.
The old client ran `collection('posts').orderBy('date','desc')` for everyone,
which (under `published == true`-gated read rules) would still be rejected
for visitors, because it could return drafts.

`loadBlogPosts()` in `index.html` now:

- visitors / anonymous: `collection('posts').where('published','==',true)` —
  permitted by the rules; sorted by `date` client-side so **no composite index
  is needed** (equality on `published` + `orderBy('date')` would otherwise
  require one).
- admin: full collection with `orderBy('date','desc')` (drafts included).

The listener is re-subscribed when auth state resolves
(`onAuthStateChanged` → `loadBlogPosts()`), so signing in as admin swaps in
the full view without a reload.

Also included: the feedback save fallback now toasts
"Saved locally — cloud unavailable" instead of failing silently after the
success toast (the data-loss-masking part of #3).

## Verify after deploying

Unauthenticated REST probes (these are the acceptance probes from #12):

```bash
# 1. Published posts readable anonymously -> 200 (was 403)
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://firestore.googleapis.com/v1/projects/ippevent-6403b/databases/(default)/documents/posts"

# 2. Feedback readable anonymously -> 200 (was 403)
curl -s -o /dev/null -w "%{http_code}\n" \
  "https://firestore.googleapis.com/v1/projects/ippevent-6403b/databases/(default)/documents/feedback"

# 3. Valid anonymous feedback create -> 201
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"fields":{"id":{"stringValue":"fb-probe"},"name":{"stringValue":"probe"},"message":{"stringValue":"probe message"},"ratings":{"mapValue":{"fields":{"overall":{"integerValue":"5"}}}},"date":{"integerValue":"1700000000000"}}}' \
  "https://firestore.googleapis.com/v1/projects/ippevent-6403b/databases/(default)/documents/feedback"

# 4. Oversized/extra-field payload -> 403 PERMISSION_DENIED (spam floor intact)
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"fields":{"id":{"stringValue":"fb-probe"},"name":{"stringValue":"probe"},"message":{"stringValue":"probe"},"ratings":{"mapValue":{"fields":{}}},"date":{"integerValue":"1700000000000"},"extra":{"stringValue":"nope"}}}' \
  "https://firestore.googleapis.com/v1/projects/ippevent-6403b/databases/(default)/documents/feedback"

# 5. Anonymous post write still denied -> 403 PERMISSION_DENIED
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"fields":{"title":{"stringValue":"nope"}}}' \
  "https://firestore.googleapis.com/v1/projects/ippevent-6403b/databases/(default)/documents/posts"
```

In-browser acceptance (from #12):

1. Incognito, no login: Blog lists published posts; feedback wall shows
   existing messages; submitted feedback survives a reload from a different
   device.
2. Incognito: an admin draft does NOT appear in the blog list.
3. Signed in as admin: editor, publish, edit, delete all work; drafts visible.
4. An authenticated *non-admin* account can neither write posts nor
   update/delete feedback (rules gate on the admin email, not on
   `request.auth != null`).

## Rollback

The previous (deny-all-anonymous) rules were the default-locked state; to
revert, restore the rules text from the Firebase console's version history
(console → Firestore → Rules → history) or `git revert` this commit and
re-deploy. No data is migrated or deleted by a rules deploy — it is
metadata-only and takes effect within ~a minute.

## Security notes

- No composite index is required by the new client query shape (see above).
- `isAdmin()` matches on the account email; Firebase Auth will not allow a
  second, unverified account to be created with that same email, so the
  email-equality check cannot be satisfied by a duplicate registration.
- If public feedback spam ever becomes a problem, the next hardening step is
  Firebase App Check on the `feedback` create path — do not loosen the
  field-set validation instead.
