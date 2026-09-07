# 9 Deck Scorepad

A card-game scorepad for 9 Deck. One person keeps score for each team, on their own phone, and both see the same game live.

Built to the same playbook as the Grocery-List app: a static page on GitHub Pages talking directly to a Firebase Firestore document, no build step, no app store, no accounts for the people using it.

---

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app. |
| `sw.js` | Service worker. Without it the app will not open with no network. |
| `manifest.webmanifest` | Makes it installable to a home screen. Upload alongside. |
| `icon-*.png`, `icon.svg`, `icon-mark.svg` | App icons, cut from the box-lid mark. Upload alongside. |
| `firestore.rules` | Security rules — paste into the Firebase console. |
| `test-scorepad.js` | jsdom harness. Run before any change. |
| `make-icons.py` | Regenerates every icon from `mark-paths.json`. |
| `build-mark.py`, `mark-paths.json`, `9_Deck.dxf`, `9_Deck_2.dxf`, `lid-source.png` | Rebuilds the mark from the box-lid artwork. Only needed if the lid changes. |

Upload `index.html`, `sw.js`, `manifest.webmanifest` and all the `icon-*.png` files to the **repo root**. The manifest and the service worker reference files by bare filename, so they all have to sit next to each other.

---

## Setup, once

**1. Make the repo.** New public repository on GitHub, separate from `Grocery-List`. Something like `9-Deck-Scorepad`. Settings → Pages → deploy from `main`, root folder.

**2. Make the Firebase project.** Console → new project → Firestore Database → create in production mode. Then Project settings → Your apps → register a web app, and copy the config object it gives you.

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
itself. One tap, no trip to Setup, and the lane is remembered for next round.

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
  the history and a fresh round opens on all four phones.

Closing runs as a Firestore transaction. Several phones may notice "both committed"
at the same moment; the first transaction to land clears the flags, and the rest
re-read, see them gone, and do nothing. No duplicate rounds.

If a phone is scoring both teams, it gets a plain **Save round** button instead,
which commits both sides at once.

---

## Offline

`sw.js` caches the app shell, icons, fonts and the Firebase library on first load,
so the app opens with no network at all. Firestore's own cache covers the data once
it is running; the service worker is what gets it running.

Two things follow from that:

- **Bump `VERSION` at the top of `sw.js` on every deploy.** The old cache is
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

Firestore's offline cache is enabled, so a dead connection is not a failure mode. Entries are saved on the phone and sent when the connection returns. The status line at the top of the Hand tab says which state you are in:

- **green** — live with the other scorer
- **amber** — no connection, saved here, will sync
- **grey** — no database configured, this phone only

The retry logic the shopping list originally needed isn't here. Firestore's own queue does it.

---

## Fixing a mistake in a saved round

Open the round in History and choose **Reopen for editing**. It comes off both
teams' totals and goes back to the Round tab, and saving it again puts it at the
end of the history — so later rounds shift up a number. It only works when nothing
is part-entered, because there is a single current round shared across every phone.

## Making changes

Run the harness first:

```
npm install jsdom
node test-scorepad.js
```

It loads the real page into a headless DOM with `localStorage` and the native dialogs mocked, drives the actual UI, and checks the scoring against the worked example. It deliberately throws if anything calls `confirm()`, `alert()`, or `prompt()` — those are silently swallowed inside Claude's Artifact sandbox, which is how the "Start new list" bug happened on the shopping list.

Then: Claude hands back the updated file, you drag it into GitHub over the old one, same filename, commit.

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

Inline copies of the mark in the app carry explicit `width` and `height`
attributes. An inline SVG sized only by CSS height with `width: auto` is laid out
inconsistently as a flex item, and gets clipped.

Re-run only if the lid artwork changes:

```
pip install pillow numpy scipy scikit-image cairosvg
python build-mark.py 9_Deck.dxf 9_Deck_2.dxf lid-source.png
python make-icons.py
```

Below about 48px the DECK band turns to mush, so the favicon sizes and the app's
own header run the 9 alone. `mark-9.svg` is that variant.

---

## Before you rely on it

The sync layer has never touched a real Firestore — the 64 checks all run in a
headless DOM with no network. The logic is exercised, the wiring is not. Before a
game night that depends on it: open the live URL on two phones, set them to
different teams, enter cards on one and watch them appear on the other, then commit
each side and confirm the round closes on both.

## Known trade-offs

- **The repo is public.** GitHub Pages' free tier requires it. Fine for card scores.
- **No authentication on the database.** Anyone with the URL and config can write to the one document. Same call as the shopping list.
- **Fonts load from Google.** Fraunces and Public Sans, matching the grocery app. At a house with no WiFi they fall back to system faces — the layout holds, it just looks plainer.
- **Sync will not work inside Claude's Artifact preview.** That sandbox blocks all outbound network calls. The app detects this and runs in single-phone mode. Test sync on the real GitHub Pages URL.
