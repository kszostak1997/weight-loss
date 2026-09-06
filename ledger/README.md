# Ledger

A calorie ledger built around one idea: the day's number should be impossible to ignore.

## Run it on your phone right now

1. Put these files on any static host (Cloudflare Pages works, `wrangler pages deploy .`).
2. Open the URL in Chrome on Android.
3. Menu → **Add to Home screen**. It installs as a standalone app, works offline, no browser chrome.

Locally: `python3 -m http.server 8080` then open `http://<your-lan-ip>:8080` on the phone.
The service worker needs `http(s)`, not `file://`.

## Files

| File | What it is |
|---|---|
| `index.html` | The whole app, including the embedded Roboto webfont. No build step, no dependencies, no network calls. |
| `manifest.webmanifest` | Makes it installable as a standalone app. |
| `sw.js` | Offline cache, network-first so redeploys are picked up. |
| `icon-*.png` | Home screen icons. |

## Screens

Five tabs: **Today** (the gauge and today's log), **Activity** (the conditioning
plan, steps, custom exercises), **Calendar**, **Progress**, **Favourites**.

Progress is also where your body stats, weigh-ins, pace and protein target live —
there's no separate settings screen. The **Body** card (age, height, sex,
starting/target weight) is the one exception to "everything's always visible":
before a profile exists it's shown as an open form; once saved it collapses to a
summary with a **Change initial setup** button, so the numbers you basically never
touch again don't sit at the top of your progress screen every time. Weigh-ins,
Pace and Protein stay fully visible underneath, right after the projected-weight
chart — those get touched often enough (or need to be visible while you're still
filling in the Body form for the first time) that collapsing them wouldn't help.

Favourites holds exactly what its name says — reusable meals and reusable
exercises — plus **Data** (export/restore/erase) at the bottom, since it has to
live somewhere and this is the closest thing left to a settings screen.

## Colour

One rule: **colour is reserved for verdicts, everything else is ink.**

A verdict is anywhere the app says good or bad — and every verdict is judged against
**maintenance**, not against the (often much more aggressive) pace goal. Eating past
your chosen pace budget but still under maintenance is still a deficit, so it still
reads green; red is reserved for actually eating back the whole day's deficit. That
is the gauge, the verdict pill, the calendar day fills, the 14-day chain, and the
"body fat moved" figure. Buttons, nav, the streak pill, "left today" and all chrome
are near-black and grey, so nothing competes with the gauge.

| Token | Value | Used for |
|---|---|---|
| `--ink` | `#16181D` | text, primary button, active nav, streak |
| `--ink-2` | `#5C636E` | secondary text |
| `--ink-3` | `#98A0AC` | labels, units, muted numerals |
| `--line` | `#E5E7EC` | card borders, dividers |
| `--track` | `#E7E9ED` | gauge track, empty calendar days |
| `--neutral` | `#F3F4F6` | the "left today" tile, icon wells |
| `--good` | `#17864F` | under maintenance |
| `--warn` | `#B0710A` | within 10% of maintenance, over the pace goal but not maintenance, and eating far too little |
| `--over` | `#C0392C` | over maintenance |

There is no brand hue and no fifth status hue. Under-eating shares amber rather than
introducing blue, since it is a rare state and a second warning rather than a third verdict.

## Typography

Roboto Variable is embedded in `index.html` as base64 (latin + latin-ext, so Polish
diacritics render properly). Four weights are used and nothing else:

| Weight | Used for |
|---|---|
| 400 | body prose, notes, empty states |
| 500 | secondary text, units, inactive controls |
| 700 | values, headings, buttons, active controls |
| 900 | hero numerals only (the gauge figure, the "to go" figure) |

## Where the data lives

`localStorage` on the device, under the key `ledger-v2`. Nothing leaves the phone,
no account, no server. Favourites → **Export a backup file** writes a JSON you can restore later.
Clearing Chrome site data wipes it, so export before you do that.

## The maths

Weight is simulated forward from what you log, day by day. Weigh-ins are optional —
log one whenever you want, and it snaps that projection back to reality. Once
today's weigh-in exists, both "Log today's weight" buttons (Today and Progress) grey
out and show the logged number instead, so there's no way to accidentally log two
for the same day; deleting it from Progress's weigh-in list re-enables them.

- **BMR** — Mifflin-St Jeor, recalculated every simulated day from the current projected weight.
- **Maintenance (TDEE)** — BMR × a fixed sedentary multiplier (`BASE_ACTIVITY = 1.2`), plus
  whatever exercise and steps were checked off in the Activity tab for that specific day
  (see below). There's deliberately no "how active are you" dropdown — that used to double-count
  training you'd then also log. Activity is real numbers you check off, not a vibe.
- **Daily budget** — maintenance minus your chosen pace, at 7700 kcal per kg.
  Held at a floor of 1500 kcal (male) / 1200 kcal (female) whatever the pace asks for.
- **Projected weight** — starting weight, then for each logged day
  `weight -= (maintenance - intake) / 7700`. Maintenance is recomputed from the new weight,
  so the budget drifts down as the projection drops, the same way it does in real life.
  If a real weigh-in exists for a day, the projection snaps to that number instead and
  keeps projecting forward from there — the deficit-bank stats stay purely food-log driven,
  only the weight trajectory gets recalibrated.
- **Protein target** — bodyweight × your chosen g/kg (Progress → Protein). Meals and
  favourites can carry a protein figure, and the Today screen shows a simple bar
  against that target — more is never penalised the way excess calories are.
- **To go** — projected weight now, minus target weight.

Because it is a model and not a measurement, drift is expected. The 7700 kcal per kg
figure is an average, and under-counted portions push the projection optimistic.
Treat the direction of the curve as the signal, not the third decimal. A weigh-in is
the fastest way to correct that drift when it matters to you.

## Activity tab

The Activity tab holds a weekly conditioning plan (currently a tennis-specific one)
and a daily step count. Checking an exercise or hitting your step goal adds real
calories to *that day's* maintenance and budget — the same way logging exercise
works in most fitness apps — so a hard training day genuinely earns you more food
without throwing off your pace. A 7-day strip lets you switch which day you're
looking at, so you can check off something from yesterday you forgot.

The fixed plan lives in one place in `index.html`: the `PLAN` constant near the top
of the `<script>` block. It's a plain object keyed by weekday, and it's meant to be
edited directly (by asking an AI assistant, or by hand) whenever the conditioning
plan changes — nothing else in the file needs to change when it does. Each exercise
has a stable `id` (don't reuse an id for a different exercise — it's what individual
days' checkmarks are keyed on), a rough `kcal` estimate at a 75&nbsp;kg reference
weight (scaled automatically to the user's current weight), and an optional `video`
URL — when present, a small ▶ link appears next to the exercise, pointing at a
YouTube demonstration, without toggling the checkbox when tapped. Every current
exercise has one filled in; it's fine to leave `video` unset on new items until
there's a good link for them.

**Favourite exercises** can be logged two ways. Most are a single fixed amount
(an evening walk, say) — tap it and it's logged. Ones marked **"Log by count"**
(Favourites → Favourite exercises) instead store a kcal-per-unit figure and a unit
label (e.g. `0.5` kcal per rep) — tapping one opens a small sheet to type how many
you actually did (28 pull-ups → 14 kcal), rather than logging a fixed guess.

On top of the fixed plan, **custom exercises** are free-form: "+ Add exercise" logs
something one-off to whichever day is selected, with its own name and kcal (given
directly, like a meal — not rescaled by bodyweight the way PLAN items are). Checking
"save to favourite exercises" keeps it in Favourites → **Favourite exercises**, so it's a
single tap to log again later — the same favourite/new-entry pattern the food side
already uses.

## Food categories

Favourites (the Favourites tab and the Add-meal sheet) can carry any combination of
`categories` — Breakfast, Main, Snacks, Drinks — since plenty of foods are more
than one thing (yoghurt is both a snack and a breakfast). A chip row above the
list filters by a single category at a time, for picking something faster on a
typical day; the favourite itself can still match several chips. Leaving every
category unchecked when saving falls back to Main. Meals logged to the day's
ledger don't carry categories themselves — only the reusable favourites do,
since that's what the chips are filtering.

## Turning it into a Play Store APK

Wrap it with Capacitor:

```bash
npm i -D @capacitor/cli && npx cap init ledger com.you.ledger
npm i @capacitor/core @capacitor/android
mkdir www && cp index.html manifest.webmanifest sw.js icon-*.png www/
npx cap add android && npx cap sync && npx cap open android
```

Set `webDir: 'www'` in `capacitor.config.json`. You get a real APK with the same code.
Bedrock is unchanged, so a PWA and the APK share one codebase.
