# Watchword

**Subtitles in your language on Netflix and YouTube** — or both languages at
once, with a dictionary card on any word and a spaced-repetition library behind
it.

A Manifest V3 Chrome extension. No accounts, no analytics, and no server of its
own: everything a user saves lives in their own browser.

`5,856 lines shipped` · `1,697 more in tests and tooling` · `9 suites, 313 assertions, zero dependencies` · `3 UI languages` · `13 translation languages`

---

## What this repository is

**A case study, not the source.** Watchword is a commercial product and its code
is closed, so publishing it here would leak the payment configuration and the
paid-tier logic along with it.

What follows is the part that is actually worth reading anyway: the problems
that were hard, the decisions that were not obvious, and the two experiments
that changed my mind. Code appears as short excerpts where a sentence of prose
would be vaguer than five lines of the real thing.

---

## The constraint that shaped everything

Watchword has no backend. That was a product decision — "nothing is tracked" is
a promise you can only make if there is nowhere for the data to go — and it
turned into the main engineering constraint:

- **No server means no place to hide.** Translation caching, licence
  validation, the review scheduler and the word library all run in the browser,
  inside a Manifest V3 service worker that Chrome stops whenever it feels like
  it.
- **The extension is a guest on someone else's page.** Netflix and YouTube
  re-render their players whenever they like, ship their own CSS, and never
  agreed to host anything.

Most of what follows is a consequence of one of those two facts.

```
                 ┌───────────────────────────────┐
  player DOM ───▶│ content.js  (per-tab overlay) │
  subtitle node  │  • platform adapter           │
                 │  • renders the subtitle layer │
                 └───────────────┬───────────────┘
                                 │ chrome.runtime message
                                 ▼
                 ┌───────────────────────────────┐
                 │ background.js (service worker)│
                 │  • all network calls          │
                 │  • one shared cache           │
                 └───────────────────────────────┘
```

Content scripts make no network calls at all. Everything goes through the
worker — partly so the streaming site's Content Security Policy cannot block
us, partly so there is exactly one cache instead of one per tab.

---

## Six problems worth explaining

### 1. "It only works after a hard refresh"

The oldest bug in the product, and the one that taught me the most.

Subtitles would sometimes not appear until the user force-reloaded the page.
Every node the extension injects is positioned relative to the player, so the
code checked that it was still on the page before re-attaching:

```js
if (!document.body.contains(el)) container.appendChild(el);
```

That check answers **"is this node in the document"**. The question that
mattered was **"is this node in the right parent"** — and the two come apart
exactly when Netflix re-renders its player subtree. Once the overlay was moved
under the wrong ancestor, `contains()` stayed `true` forever, so nothing ever
put it back. An absolutely-positioned overlay under a `position: static`
ancestor renders off-screen, which is why the symptom was "no subtitles" rather
than "subtitles in a funny place".

```js
function attach(el, container) {
  if (el.parentElement !== container) container.appendChild(el);
  return el;
}
```

Two related decisions came out of the same investigation:

- **Never fall back to `document.body`.** The old code did, and attaching to
  the wrong element is worse than not attaching at all: it looks like success,
  so nothing ever retries.
- **Wait for both elements, not one.** On Netflix the node we observe
  (`.player-timedtext`) and the node we attach to (`.watch-video`) are
  different, and the subtitle node usually appears first. Starting on that
  alone was what put the overlay under the wrong ancestor in the first place.

A one-second health check now re-checks the parent, and the test suite proves
the specific case: a node that is still `isConnected === true` but under the
wrong parent gets moved back.

### 2. The subtitle bug that was a CSS bug

A user reported that translations came out in ALL CAPITALS. My first diagnosis
was that the subtitle track itself was uppercase and the translation engine was
preserving it — plausible, partly true, and **not the cause**.

`text-transform` is an inherited property. The overlay is injected *inside* the
site's player container, and nothing in the stylesheet reset it. A player that
upper-cases its own caption layer silently upper-cases ours too.

The lesson generalises: **an injected node inherits everything the host page
hands down**, and a styling bug that arrives through inheritance looks like a
bug in whatever the text went through last. This one cost hours in the
translation pipeline before I thought to look at CSS.

```css
#ww-overlay, #ww-popup, #ww-hint {
  /* Refuse whatever the page was about to hand down. */
  text-transform: none;
  font-style: normal;
  font-variant: normal;
  letter-spacing: normal;
  word-spacing: normal;
  text-indent: 0;
}
```

A test now asserts that all three injected roots declare these, because the only
way they disappear is someone tidying the stylesheet — and they are invisible
until the day they matter.

### 3. Translate sentences, not subtitle cues

A subtitle cue is a display unit, not a sentence. "Ich habe gestern" and "mit
ihm gesprochen." arrive as two separate cues, and translating each on its own is
the single biggest reason machine-translated subtitles read badly: the engine is
being asked about half a clause.

Watchword holds a cue that does not end in punctuation and prepends it to the
next one, so the endpoint sees a whole sentence. **This costs no extra
requests** — each cue still triggers exactly one; the later ones simply carry
more.

I did not want to claim it worked without evidence, so I measured it. Eight
sentences split the way a cue would split them, translated both ways:

| translated cue by cue | translated as one sentence |
|---|---|
| "dün yaptım onunla konuştu." | **"Dün onunla konuştum."** |
| "Bana söyledi gelemeyeceğini." | **"Bana gelemeyeceğini söyledi."** |
| "Ayakta duran kadın kapının yanında annem var." | **"Kapının yanında duran kadın benim annemdir."** |

**8 of 8 differed**, and the left column is not merely clumsier — the first row
is not grammatical Turkish at all. The measurement lives in the repo as a
runnable script, so the claim can be re-checked if the engine changes.

Two caps keep the buffer honest, and both come from real footage rather than
imagination: YouTube's auto-captions frequently contain **no punctuation at
all**, so the string would otherwise grow for the length of the video; and a gap
longer than five seconds means a seek or a scene change, so whatever comes next
is not a continuation.

That gap is measured in **video seconds, not wall-clock seconds** — a detail I
got wrong first time. The extension has an auto-pause feature that stops the
video on every subtitle line, so a careful viewer can sit on one cue for a
minute. On a wall clock every gap looked like a scene change, which switched
sentence joining off for exactly the users most likely to want it.

### 4. The experiment that deleted a feature

The obvious next step was a context window: send the *previous* sentence along
with the current one so the engine can resolve pronouns and formality — German
`sie` is "she", "they" or formal "you", and Turkish distinguishes the last one.

I designed it, wrote the plan, and then measured before building.

```
context changed the output in 0 of 12 cases
```

Twelve pairs: two source languages, both newline- and space-joined, textbook
ambiguities (*Bank* = bench/bank, *Schloss* = castle/lock, *bat*, *crane*), and
formal versus informal address. The endpoint does not look at what precedes a
sentence. **Its context window is one sentence.**

So the feature was never written. It would have added complexity to the
translation path, cut the cache hit rate (context becomes part of the key), and
delivered nothing.

The same measurement explains why the *previous* feature works: the boundary is
the sentence, so completing a sentence helps and reaching past it does not.
Both probes ship as a script whose expected result for the context mode is
**zero** — if a future run returns anything else, the engine has changed and the
idea is worth revisiting.

I think this is the part of the project I would most want to be judged on. The
feature that does not exist took more discipline than the four that do.

### 5. Making the release checklist executable

Shipping to the Chrome Web Store has a list of things that must be true, and two
of them are the kind that only hurt after the fact:

- a development flag that, if left on, hands **every user a one-click paid
  unlock**
- a payment configuration that, if pointed at the sandbox, fails **only for
  customers who have already paid**

A checklist in a markdown file does not stop either. So the build refuses:

```
$ node build.js --release
NOT READY TO SHIP:
  - WATCHWORD_POLAR_ENV = "sandbox" — must be production
  - production.orgId is empty
  - manifest.json still carries the sandbox host permission
```

The package's file list is **derived** from the manifest rather than maintained
by hand — every page the extension opens, the fonts referenced by the
stylesheets, the service worker's `importScripts`. The previous hand-written
list had gone stale and was missing five files including the translation module,
which produced a package that installed perfectly and then translated nothing.
A test now locks the derivation.

The same idea produced my favourite small test in the project: **every host
permission in the manifest must appear in both privacy documents.** Requesting a
permission you do not disclose is precisely what store review looks for, and I
had already drifted once.

### 6. Reading what an API actually does

A licence key bought in the sandbox refused to activate:

```
NotPermitted — "This license key does not support activations."
```

The key was fine. The payment provider refuses `/activate` entirely when the
benefit has no activation limit configured — with no limit there is no slot to
take, so validation is the only check it offers.

The part that mattered: **that setting is copied onto each key when the key is
granted.** Turning the limit on later does not rescue keys already sold. So the
fix could not live in a dashboard; it had to be in the client, which now falls
back to validation for exactly that error.

Diagnosing it meant reading the provider's responses rather than its
documentation. An invented key returned `Not found` while the real one returned
a different message — which proved the organisation ID was right and narrowed
the problem to the key's own state.

---

## Testing

Nine suites, 313 assertions, **zero dependencies**. `node test/run.js` works on a
clean checkout with nothing installed.

The suites drive the real source against small hand-written fakes: `content.js`
runs in a fake DOM, `settings.js` against a fake `chrome.storage`. Every suite
exists because something went wrong once:

| suite | what it locks |
|---|---|
| `attach` | the parent-vs-document bug, and what each subtitle mode draws |
| `sentence` | cue joining — punctuation, caps, seek detection |
| `review` | spaced-repetition scheduling and retirement |
| `studio` | the customisation contract and its injection boundary |
| `pages` | page wiring, and that every UI string is both defined and used |
| `package` | that the shipped package is complete and discloses its permissions |
| `backup` · `stats` · `speed` | merge rules, streak arithmetic, rate normalisation |

One test taught me something about testing itself. A cap in the sentence buffer
had a bug: once hit, joining switched off for the rest of the video. The
existing test asserted "the buffer does not grow" — which passed, because a
buffer that is switched off does not grow either. **The test was true and
useless.** Its replacement asserts that joining resumes *after* the cap.

Customisation values are validated as a **security boundary rather than a
nicety**: they end up in `style.setProperty` on a third-party page, so numbers
are clamped and colours must match `/^#[0-9a-f]{6}$/`. The suite keeps the
injection cases.

---

## Design decisions

**Two colour systems that never meet.** Tier identity (free/paid) lives in the
extension's own chrome; word-progress state lives in content surfaces. They are
deliberately different hues, so gold never has to mean both "lifetime" and
"learning" in the same glance.

**Weight, not colour.** The review buttons are told apart by ghost → tinted →
solid → outlined, plus distinct glyphs. The first version used adjacent hues
that were nearly identical under deuteranopia. If a distinction disappears in
greyscale, it was never a distinction.

**Three durations, no more.** 120ms confirms an input, 180ms a state change,
90ms an appearance. Subtitle lines get **zero** — a line changes every one to
three seconds, and animating each change is an interface that flickers
continuously. `prefers-reduced-motion` is honoured without exception: the video
behind the UI is already a moving background.

**The customisation preview is the product.** The editor page loads the real
overlay stylesheet and applies settings through the same function the video
does, so the preview cannot drift from the result.

---

## What is not finished

- **Store submission.** The build gate still lists the production payment
  configuration as outstanding.
- **Screenshots.** The ones in the product repo show a previous interface. They
  have to be retaken while a development flag is still on, because the tier
  switcher depends on it — a sequencing detail that is easy to discover too
  late.
- **More platforms.** The adapter interface is eight methods and adding a site
  is mostly writing one; the cost is not code but that each new host permission
  is another review round.
- **A second translation engine.** Now that translation is the product's core
  rather than a study aid, depending on a single unofficial endpoint is a
  strategic risk more than a quality one.

---

## Elsewhere

The extension itself is on the Chrome Web Store.

Questions about the work here — the architecture, the reasoning behind a
decision, or the parts I got wrong before I got them right — are welcome as an
issue on this repository, or through [github.com/ertanm](https://github.com/ertanm).
