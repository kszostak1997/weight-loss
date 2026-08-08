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

## Colour

One rule: **colour is reserved for verdicts, everything else is ink.**

A verdict is anywhere the app says good or bad. That is the gauge, the verdict pill,
the calendar day fills, the 14-day chain, and the "body fat moved" figure. Those get
green, amber or red. Buttons, nav, the streak pill, "left today" and all chrome are
near-black and grey, so nothing competes with the gauge.

| Token | Value | Used for |
|---|---|---|
| `--ink` | `#16181D` | text, primary button, active nav, streak |
| `--ink-2` | `#5C636E` | secondary text |
| `--ink-3` | `#98A0AC` | labels, units, muted numerals |
| `--line` | `#E5E7EC` | card borders, dividers |
| `--track` | `#E7E9ED` | gauge track, empty calendar days |
| `--neutral` | `#F3F4F6` | the "left today" tile, icon wells |
| `--good` | `#17864F` | under budget |
| `--warn` | `#B0710A` | within 10% of the line, and eating far too little |
| `--over` | `#C0392C` | over budget |

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
no account, no server. Setup → **Export a backup file** writes a JSON you can restore later.
Clearing Chrome site data wipes it, so export before you do that.

## The maths

There are no weigh-ins. Weight is simulated forward from what you log.

- **BMR** — Mifflin-St Jeor, recalculated every simulated day from the current projected weight.
- **Maintenance (TDEE)** — BMR × activity multiplier (1.2 to 1.9).
- **Daily budget** — maintenance minus your chosen pace, at 7700 kcal per kg.
  Held at a floor of 1500 kcal (male) / 1200 kcal (female) whatever the pace asks for.
- **Projected weight** — starting weight, then for each logged day
  `weight -= (maintenance - intake) / 7700`. Maintenance is recomputed from the new weight,
  so the budget drifts down as the projection drops, the same way it does in real life.
- **To go** — projected weight now, minus target weight.

Because it is a model and not a measurement, drift is expected. The 7700 kcal per kg
figure is an average, and under-counted portions push the projection optimistic.
Treat the direction of the curve as the signal, not the third decimal.

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
