# Sentinel AI · URL Scanner

A single-file, 100% client-side URL threat scanner. Paste in a link and it
runs a deterministic, weighted heuristic engine over the URL's structure —
domain reputation, structural anomalies, lexical content, and
encoding/obfuscation tricks — the same class of signals browsers and mail
filters check before a page ever loads. Nothing leaves the browser; there
is no backend, no network call, no build step.

## Quick start

Just open `index.html` in any modern browser. That's it — no install,
no server, no dependencies.

```bash
open index.html      # macOS
# or double-click the file / drag it into a browser tab
```

## What it does

Type or paste a URL into the scan bar (or click one of the example chips,
or upload a QR image) and hit **SCAN**. Sentinel will:

1. Parse the URL and normalize it (adding `http://` if no scheme is given).
2. Run it through four weighted detection models (below).
3. Render an overall risk score (0–100), a verdict (SAFE / SUSPICIOUS /
   DANGEROUS, etc.), a plain-English AI-style summary, a per-category
   breakdown, and a detailed list of every triggered finding with its
   severity and point value.
4. Explain **attacker intent** instead of just saying "malicious" —
   credential harvesting, brand impersonation, malware delivery,
   redirect laundering, or filter evasion, each with confidence + why.
5. Tell a plain-English **attack story**, flag **brand impersonation**
   (claims-vs-actual), warn on **credential traps**, and show a
   **risk heatmap** of which URL parts look bad.
6. Show a **safe preview** (URL anatomy, nothing fetched), **predicted
   page behaviors**, and an **evidence timeline** of what ran in order.
7. Log the scan to a "recent" history strip you can click back into, and
   track **per-URL risk history** in `localStorage` (this browser only).
8. Let you copy the full report as plain text via the **COPY** button
   (now includes intent, story, impersonation, and behaviors).

## Detection models

| Category | Examples of what it flags |
|---|---|
| **Domain Reputation** | Typosquatting / lookalike brand names (Levenshtein distance), suspicious TLDs (`.tk`, `.xyz`, `.top`, …), known URL shorteners |
| **Structural** | Dangerous schemes (`javascript:`, `data:`, `file:`), raw IP-address hosts, no HTTPS, `@` tricks, excessive length, excessive hyphens/subdomains |
| **Lexical / Content** | Credential-harvesting keywords (`login`, `verify`, `account-suspended`, …), dangerous file extensions (`.exe`, `.apk`, `.hta`, …) |
| **Encoding / Obfuscation** | Hex/octal/decimal-encoded IPs, punycode/homograph domains, mixed-script hostnames, high-entropy base64-looking query params, open-redirect parameters |

Each finding carries a point weight and a severity (`low` / `medium` /
`high`); totals are summed per category and combined into the overall
score and verdict.

> **Scope & honesty note (also stated in the app footer):** this tool
> does *not* fetch the live page (browser CORS won't allow it from a
> static file) — it only analyzes URL *structure*. The "AI Analysis"
> text is template-generated from the triggered signals, not output from
> a live ML model. It's a rule-based triage/teaching tool, not a
> replacement for a maintained threat-intel feed.

## Interface

- **Scan bar** — paste a URL, press Enter or click SCAN.
- **QR → URL → Risk** — "Scan QR image" decodes a QR photo locally
  (native `BarcodeDetector`) and auto-scans the URL. Works in
  Chrome/Edge; other browsers get a notice to paste the link manually.
- **Example chips** — one-click samples covering typosquatting, raw/hex
  IPs, punycode, shorteners, hyphenated fake-brand domains, and
  open-redirect params.
- **Recent history** — the last several scans, color-coded by severity;
  click a chip to re-run that scan.
- **Score gauge** — an animated ring + counting number (0–100) that
  eases into place in sync with the ring's fill animation.
- **Phishing Intent Engine** — intent cards with confidence
  (credential harvesting, brand impersonation, malware, redirect
  laundering, filter evasion).
- **Attack story + credential warning + impersonation card** —
  plain-English "what could happen", a do-not-log-in banner on
  password-theft patterns, and a claims-vs-actual brand comparison.
- **Risk heatmap** — URL split into scheme/subdomain/domain/TLD/port/
  path/query tokens, colored high/med/ok.
- **Safe preview** — URL anatomy table (protocol, host, path, query,
  redirect target). Nothing is fetched or rendered.
- **Predicted behaviors + evidence timeline** — heuristic behavior
  predictions (fake form, forced download, redirect hop, …) and the
  ordered check stages with hit counts.
- **URL risk history** — per-URL trail (`localStorage`, this browser
  only): first → now, last-change delta, full score trail.
- **Category bars** — four bars showing how much each model contributed
  to the score.
- **Findings list** — every triggered check, its severity badge, and its
  point contribution.
- **Copy report** — copies a plain-text summary of the current scan to
  the clipboard.

## Tech

- Plain HTML/CSS/vanilla JavaScript — no frameworks, no build tooling,
  no external JS dependencies.
- QR decoding uses the native `BarcodeDetector` API where available —
  no library shipped. Deliberately **no OpenCV**: it adds ~8MB for a
  problem native APIs + (optionally) a ~50KB jsQR fallback solve.
  OpenCV only makes sense later for damaged/blurry QR preprocessing.
- Google Fonts (`JetBrains Mono`, `Inter`) loaded via `@import` for the
  terminal/console aesthetic.
- `localStorage` key `sentinel-risk-history-v1` stores per-URL score
  trails (this browser only, last 12 scans per URL).
- Everything (styles, data tables, detection logic, UI rendering) lives
  in the one `index.html` file.

## File structure

```
index.html   – everything: markup, styles, and the scanning engine
```

## Customizing

All of the reference data the engine checks against lives at the top of
the `<script>` block, so it's easy to extend:

- `KNOWN_BRANDS` — brand names used for typosquat comparisons
- `SUSPICIOUS_TLDS` — TLDs weighted as higher-risk
- `URL_SHORTENERS` — known shortener domains
- `SUSPICIOUS_KEYWORDS` — credential-harvesting / urgency keywords
- `DANGEROUS_EXTENSIONS` — risky file extensions
- `REDIRECT_PARAMS` — query-param names checked for open-redirect abuse

The core scoring logic lives in `analyze(rawUrl)`; intent/story/heatmap/
timeline/behavior builders (`buildIntents`, `buildAttackStory`,
`buildHeatTokens`, `buildTimeline`, `buildBehaviors`,
`buildCredentialWarning`) sit right after it, and all DOM rendering lives
in `renderReport()` + the `renderX()` helpers.

## Disclaimer

Sentinel AI is a static-analysis / educational triage tool. It does not
fetch or render the target page, does not call any external API or
threat-intel service, and should not be used as a sole basis for
security decisions.