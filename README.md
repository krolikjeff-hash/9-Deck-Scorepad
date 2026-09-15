# 9 Deck Scorepad

A card-game scorepad for 9 Deck. One person keeps score for each team, on their own phone, and both see the same game live. It will also **read a hand from a photograph** — both the books on the table and the pile of unplayed cards — so a round can be entered by taking two pictures instead of tapping a hundred times.

Built to the same playbook as the Grocery-List app: a static page on GitHub Pages talking directly to a Firebase Firestore document, no build step, no app store, no accounts for the people using it.

**Live:** `krolikjeff-hash.github.io/9-Deck-Scorepad/` — app `2026-09-15d`.

---

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app. |
| `sw.js` | Service worker. Without it the app will not open with no network. |
| `glyphs-v3.onnx` | The card detector — 12 MB, 15 classes. Reads the rank index in a card's corner. |
| `glyphs-v2.onnx` | The previous detector, 13 classes. Kept only until every phone has updated; delete it then. |
| `share.html` | A scannable card for handing the app to people at the table. Linked from Setup. |
| `make-qr.py` | Regenerates `share.html`. Only needed if the URL changes. |
| `patgen.py` | Regenerates the background pattern tile. See **The background** below. |
| `manifest.webmanifest` | Makes it installable to a home screen. Upload alongside. |
| `icon-*.png`, `icon.svg`, `icon-mark.svg` | App icons, cut from the box-lid mark. Upload alongside. |
| `firestore.rules` | Security rules — paste into the Firebase console. |
| `test-scorepad.js` | jsdom harness for the scorepad. Run before any change. |
| `test-photo-reader.js` | Playwright harness for the reader — real browser, real model, real photographs. |
| `make-icons.py` | Regenerates every icon from `mark-paths.json`. |
| `build-mark.py`, `mark-paths.json`, `9_Deck.dxf`, `9_Deck_2.dxf`, `lid-source.png` | Rebuilds the mark from the box-lid artwork. Only needed if the lid changes. |

Upload `index.html`, `sw.js`, the model, `manifest.webmanifest` and all the `icon-*.png` files to the **repo root**. The manifest and the service worker reference files by bare filename, so they all have to sit next to each other.

---

## Setup, once

**1. Make the repo.** New public repository on GitHub, separate from `Grocery-List`. Something like `9-Deck-Scorepad`. Settings → Pages → deploy from `main`, root folder.

**2. Make the Firebase project.** Console → new project → Firestore Database → create in production mode. Then Project settings → Your apps → register a web app, and copy the config object it gives you.

One project can host several small apps — the free tier is per project, and a household's traffic is nowhere near it. `firestore.rules` is written in a per-app shape for that: one top-level collection and one `match` block per app, with a default-deny at the bottom. Adding the next app is appending a block, not a restructure. Register a second web app inside the same project to get its own `appId` against the same `projectId`.

**3. Paste the config into the app.** Near the top of `index.html`:

```js
var FIREBASE_CONFIG = {
  apiKey: "",
  authDomain: "",
  projectId: "",
  storageBucket: "",
  messagingSenderId: "",
  appId: ""
};
```

Leave it blank and the app still works — it just keeps score on one phone with no sync. That is a legitimate way to use it, and a useful fallback if Firebase ever misbehaves mid-game.

**4. Publish the rules.** Firestore Database → Rules → paste `firestore.rules` → Publish.

**5. Upload.** Drag the files into the repo's upload page and commit. Live in a minute or two.

**6. Verify it actually syncs**, not just that it loads. Open the link on two phones. Set one to score Team 1 and the other to Team 2 (Setup tab). Add cards on one and watch them appear on the other. A page that loads is not the same claim as a page that syncs.

---

## Reading a hand from a photograph

Each of the two tables on the Round tab carries an **Automatic Scoring** button. Tapping it opens the reader already knowing what it is reading and for whom — the table settles books-or-unplayed, and the round's own team selection settles the team. There is no separate Photo tab and no second team picker to disagree with the first.

The flow is the same for both halves: take the photo, turn it upright, drag the four bars in to trim, read, correct anything wrong with the crosshair and the pad, apply.

- **Books** places wilds into their rank columns and adds the books to the round.
- **Unplayed** counts the pile into the six point buckets and **replaces** the round's unplayed figures rather than adding to them. The team's unplayed cards are gathered and photographed once, so a second photo is a correction, not an addition.

Applying clears the page, so the same photo can never be applied twice.

**Turning it upright is load-bearing, not cosmetic.** The detector was trained on upright glyphs with about six degrees of jitter. On a photo lying on its side it does not read badly — it goes blind: 1 card out of 77 against 77 out of 77 upright. That is why the app makes you confirm the orientation, and why a read that comes back nearly empty says so rather than shrugging.

**Trim the sides as well as the top.** A stray card lying past the last book gets read, and being read it lands in some book or other.

### How well it reads

Measured against hand-marked ground truth, not impressions:

| | |
|---|---|
| Books, round 8, through the app | 77/77 naturals, 12/12 wilds, 11 of 11 books exact |
| Unplayed, four held-out hands, 87 cards | every class exact, points off **0** |
| All 24 unplayed photographs, 486 cards, from cold | 0 missed, 0 invented, 0 points off |
| Simulated sensors, 2400–8000 px on the long side | identical results at every width |

The reader is given nothing about the photograph — not the phone, not the distance, not the card size. It normalises the frame to one canonical width, searches a ladder of read scales on a fixed slice of the frame, and picks by the glyph size the model itself reports. Every distance in it is a fraction of that measured glyph rather than a pixel count, because a pixel constant is a statement about one camera distance.

It has not yet been used at a real table, and no read has been timed on an actual phone. Sandbox numbers are roughly 6–8 seconds for a pile of unplayed cards and around 25 for a full books photo.

**The camera preview is Android's, not the app's.** `<input capture="environment">` hands off to the system capture activity, which on a Pixel shows a 4:3 preview letterboxed into a tall screen. The page cannot influence it — there is no attribute for it and no flag to find. Building an in-page camera with `getUserMedia` would fix the preview and cost the still pipeline: full sensor resolution, HDR+, the whole computational stack, replaced by a video frame at whatever resolution the device grants. Every photo the model has ever seen was a full-resolution still, so this is deliberately left alone.

---

## The background

There is no logo in the masthead. The mark **is** the background: a brick step-and-repeat of the 9, 35 px at 15% opacity, over a flat baize.

Two things about it are worth keeping straight:

- **The 9 alone, never the whole lockup.** Below about 48 px the DECK band stops being letters, and a tile is smaller than that. Tiled, the full lockup reads as dirt on the screen.
- **The pattern is generated, not hand-written.** `patgen.py` emits the tile for any layout (grid, half-drop, brick, court), size, spacing and colour. It lands in `index.html` as a single ~1.9 kB `data:` URI on `body`. **Regenerate it; do not edit that line.** It has already been broken once by hand — the `data:image/svg+xml,` prefix went missing, and a browser drops an unparseable background silently, which looks exactly like a pattern that is too subtle.

At these settings the marks blend to `#1A563A`, within a shade of `--baize-line` — the pattern and every hairline in the interface are the same green, by arithmetic rather than by aim.

The radial vignette that used to sit on `body` was removed when the pattern arrived. It darkened precisely the side gutters, which on a phone is the only place the pattern has room to show.

**To check a change to the tile, sample the pixels rather than trusting your eyes.** An empty patch of masthead returns one distinct colour when the background is broken and about thirty when it is working. At 15% the difference is too subtle to call by looking.

---

## However many people are counting

A common split is one person per team on the books and another per team on the
unplayed cards — four at once. The app supports that, but it doesn't require it.
Roles change round to round depending on who got up for a drink, so nothing here
is mandatory.

**Out of the box, every phone can enter anything.** That is the right setting for
one scorer, and a perfectly good setting for four.

Optionally, a phone picks a **lane**: a team, and a half of the count. On its own a
lane changes only two cosmetic things — which team the Round tab opens on, and
which team the sticky button commits. It does not restrict anything.

If a table wants the guard rail, **Lock this phone to its lane** makes it strict:
everything outside the lane goes visible-but-locked, so nobody enters cards in the
wrong column. When that turns out to be inconvenient — someone stepped away, and
their column needs filling — there is a **Take over** button on the locked section
itself. One tap, no trip to Setup, and the lane is remembered for next round. The
Automatic Scoring button on a locked table is disabled too, so nobody photographs
and corrects a whole hand only to be refused at the last button.

Either way the four write paths are disjoint, which is what makes concurrent
counting safe:

```
nine-decks/current
  shared.current.A.books   <- Team 1, books
  shared.current.A.left    <- Team 1, unplayed cards
  shared.current.B.books   <- Team 2, books
  shared.current.B.left    <- Team 2, unplayed cards
  shared.current.wentOut   <- whoever sets it, one field
  shared.rounds            <- closed rounds
  shared.names, shared.rules
```

Every write names the exact field it changed and sends only that field. This
matters more than it sounds: writing the whole document means each phone ships its
own copy of everyone else's fields, and the last one to write wins — quietly
undoing whatever the other three just entered.

Writes are debounced by about half a second, so tapping "+" nine times is one write.

## Committing independently

Each team commits its own side when it's done. Teams finish at different speeds, so
neither waits on the other.

Both commit buttons live in the **Finishing the round** sheet and anyone can press
either one — the person holding a phone when the counting ends should be able to
say so, whatever lane they picked. A phone with a lane also gets its own team's
commit as a shortcut in the sticky bar.

- **Commit** locks that team's books and unplayed cards for everyone, and shows a
  tick beside their score. This is the only hard lock in the app.
- **Undo commit** reopens them, right up until the round closes.
- When **both** teams have committed, the round closes on its own: it moves into
  Game Totals and a fresh round opens on all four phones.

Closing runs as a Firestore transaction. Several phones may notice "both committed"
at the same moment; the first transaction to land clears the flags, and the rest
re-read, see them gone, and do nothing. No duplicate rounds.

If a phone is scoring both teams, it gets a plain **Save round** button instead,
which commits both sides at once.

## The scoring rules are read-only

The Setup tab shows every value a score is built from, and none of them can be
edited. That is not tidiness. A saved round stores **card counts, not a score**,
and is re-scored from the rules every time it is displayed — so one person quietly
retyping a card value mid-game silently re-scores every saved round on every
phone. That is a large, invisible, shared consequence for something that looks
like a per-phone setting.

The values still live in `shared.rules` and still sync, so a backup restored from
another phone carries its own numbers and both displays and scores by them. That
is the escape hatch if a house ever plays different values.

---

## Sharing it at the table

`share.html` shows a QR code for the app, the URL in text, a copy button, and the
add-to-home-screen instructions for both platforms. The Setup tab links to it, so
the usual move is to open Setup and hand someone your phone.

The QR is generated by `make-qr.py` and baked into the page as inline SVG paths —
no script, no library, no network. Two consequences worth having: it renders on a
phone with no signal, and it prints cleanly, so it can go inside the card box lid.
Colours are set as attributes on the shapes rather than only in CSS, because a QR
that depends on a stylesheet is a black square the moment the stylesheet does not
arrive.

If the site ever moves, regenerate rather than editing the page by hand:

```
pip install segno
python make-qr.py https://the-new-url/
```

## Offline

`sw.js` caches the app shell, icons, fonts and the Firebase library on first load,
so the app opens with no network at all. Firestore's own cache covers the data once
it is running; the service worker is what gets it running.

The detector's model and its runtime are cached too, but in a cache that is
**deliberately not versioned**: a deploy must not make every phone download 12 MB
again. A retrained model gets a new file name, which is a new cache entry, and
stale entries are dropped on activate. Two deploys have now gone past that cache
without disturbing it.

Two things follow:

- **Bump `VERSION` at the top of `sw.js` on every deploy.** The old shell cache is
  dropped when the new worker activates, so a stale shell can never outlive a
  release. Forget this and people keep seeing the old app.
- **It needs HTTPS**, so it does nothing over `file://` or inside Claude's preview.
  That is expected — test offline behaviour on the real Pages URL, in an installed
  home-screen copy.

Firestore's own transport is deliberately never intercepted. It long-polls and
streams, and a cache in the middle of that breaks live sync.

The Setup tab shows the running version, which is the quickest way to confirm a
deploy actually landed on a given phone.

## Connectivity

Firestore's offline cache is enabled, so a dead connection is not a failure mode. Entries are saved on the phone and sent when the connection returns. The status line at the top of the Round tab says which state you are in:

- **green** — live with the other scorer
- **amber** — no connection, saved here, will sync
- **grey** — no database configured, this phone only

The retry logic the shopping list originally needed isn't here. Firestore's own queue does it.

---

## Fixing a mistake in a saved round

Open the round in Game Totals and choose **Reopen for editing**. It comes off both
teams' totals and goes back to the Round tab, and saving it again puts it at the
end of Game Totals — so later rounds shift up a number. It only works when nothing
is part-entered, because there is a single current round shared across every phone.

**Start a new game** clears the saved rounds. Team names and scoring rules are
deliberately kept — the same two teams usually play again.

## Making changes

Run both harnesses first:

```
npm install jsdom
node test-scorepad.js          # 152 checks, headless DOM

npm install playwright
node test-photo-reader.js      # 61 checks, real browser and real model
```

`test-scorepad.js` loads the real page into a headless DOM with `localStorage` and the native dialogs mocked, drives the actual UI, and checks the scoring against the worked example. It deliberately throws if anything calls `confirm()`, `alert()`, or `prompt()` — those are silently swallowed inside Claude's Artifact sandbox, which is how the "Start new list" bug happened on the shopping list.

`test-photo-reader.js` runs the page in a real browser against real photographs and the real model, and compares what comes out against the marked ground truth. It has caught things the offline Python holdout missed, and is the reason to keep it slow and real rather than mocking the detector.

Then: Claude hands back the updated file, you drag it into GitHub over the old one, same filename, commit. Files under 10 MB can go up through the browser bridge; the model is over that and is dragged on by hand.

### Verifying a deploy

**Do not verify by comparing bytes from a URL.** Two separate things make that unreliable, and both looked like failed deploys before they were understood:

- The **live Pages host** returned `index.html` 600 bytes longer than what was pushed, with a script tag injected before `</head>` that is not in the repo. GitHub's own `ETag` encoded the correct length and the raw copy was exact, so the rewrite was happening somewhere on the path to that one machine.
- **`raw.githubusercontent.com` lags** behind a push by minutes, so straight after a commit it happily serves the previous version with the previous hashes.

What is authoritative immediately is the repository's own commit view, and — for whether the deploy actually works — **the running app**: load the live URL, `update()` the service worker, and read `APP_VERSION`, the cache names and the DOM out of the page. That exercises what the phones will exercise, which is a better question than whether two files hash the same.

---

## The mark

The icon is the lid of the physical card box, built from the cut files rather than
reconstructed by eye. Two features of it are easy to get wrong: the 9 runs edge to
edge on its plate rather than sitting inset, and its lower counter is a tongue that
breaks out through the left side of the glyph, with a step in its lower edge — not
a rounded rectangle with a slot cut to it.

`build-mark.py` assembles it from two sources, and the split is deliberate:

- **DECK** comes from `9_Deck_2.dxf`, as quadratic splines.
- **The 9** comes from `lid-source.png`, the box-top artwork — *not* from
  `9_Deck.dxf`. The two disagree in one place: where the tongue-shaped counter runs
  toward the left edge of the glyph, the box has a large concave fillet and the DXF
  has a hard 90° corner. The DXF is an earlier revision. If a newer one turns up
  with that fillet, switch back to it — vector beats a traced screenshot.
- `9_Deck.dxf` is still read, for its plate rectangle alone, which is what puts the
  DXF-drawn DECK and the screenshot-drawn 9 into one coordinate frame.

Neither source is a filled shape — both are outlines — so what counts as stroke and
what counts as counter is a question about topology. The two halves need different
answers to it:

- **DECK** is picked out by nesting depth. Its file holds only letters, so a region
  one level in from the edge is a letter and two levels in is a counter.
- **The 9** is picked out by probing a point inside the numeral's stroke instead.
  Depth is unreliable there because the 9 runs flush to the plate, putting two
  boundaries on top of each other.

Output is flattened to within about a fifth of a pixel on a 500px-wide drawing.
Stitching the DXF entities into closed loops analytically was tried first and
abandoned: both files carry duplicate, overlapping edges, which defeats naive
endpoint walking and silently yields loops that cut diagonally across the glyph.

Below about 48px the DECK band turns to mush, so the favicon sizes run the 9
alone; `mark-9.svg` is that variant, and it is the same silhouette the background
pattern tiles. **The app's header carries no mark at all** — the background is
where the mark lives now.

Any inline copy of the mark carries explicit `width` and `height` attributes. An
inline SVG sized only by CSS height with `width: auto` is laid out inconsistently
as a flex item, and gets clipped.

Re-run only if the lid artwork changes:

```
pip install pillow numpy scipy scikit-image cairosvg
python build-mark.py 9_Deck.dxf 9_Deck_2.dxf lid-source.png
python make-icons.py
```

---

## What is proven, and what is not

- **Sync works against a real Firestore.** A round played across a PC and a phone
  on 2026-09-14 updated the score on both. That was the long-standing gap: the
  harness runs in a headless DOM with no network, so the logic was exercised and
  the wiring was not.
- **Two phones at a table, on different networks, is still unproven.** PC plus
  phone proves the Firestore end; it does not prove two mobile connections
  reconciling. Before a game night that depends on it, do the two-phone check in
  step 6 above.
- **The reader has never been used at a real table.** It is accurate on 24
  photographs of a full nine-deck shoe and on eight marked rounds, but every one of
  those was taken deliberately, in good light, by one person.

## Known trade-offs

- **The repo is public.** GitHub Pages' free tier requires it. Fine for card scores.
- **No authentication on the database.** Anyone with the URL and config can write to the one document. Same call as the shopping list.
- **Fonts load from Google.** Fraunces and Public Sans, matching the grocery app. At a house with no WiFi they fall back to system faces — the layout holds, it just looks plainer.
- **The model is 12 MB.** First load on a phone downloads it once, then it is cached across deploys. On a bad connection the first photo read will wait on that.
- **Sync will not work inside Claude's Artifact preview.** That sandbox blocks all outbound network calls. The app detects this and runs in single-phone mode. Test sync on the real GitHub Pages URL.
