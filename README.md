# URL Threat Scanner — Study Guide / README

This is a **single HTML file** that works as a complete web app — no server, no build tools, no installs. You open it in a browser and it just runs. That's intentional: everything you need to understand it is in one place.

This README walks through *every* piece of it, in plain language, in the order a beginner should learn it. Read it top to bottom once, then use it as a reference while you read the actual code side by side.

---

## 1. The big picture — what does this app actually do?

You type a URL (like `http://paypa1-secure-login.tk/verify-account`) into a box and click **SCAN**. The app does **not** visit that website. It never sends your URL anywhere — no server, no API call, nothing leaves your browser tab.

Instead, it looks at the *text* of the URL itself and checks it against a list of known phishing red flags: Is it a raw IP address instead of a real domain? Does it misspell a famous brand? Is it using a sketchy top-level domain like `.tk` or `.zip`? Does it contain words like "verify" or "urgent" that scammers love?

Each red flag adds points to a risk score (0–100). At the end, you get:
- A **score** (e.g. 71/100)
- A **verdict** (e.g. "HIGH RISK — likely phishing")
- A **list of findings** — each individual red flag, with an explanation and how many points it added

This is the same *category* of check that real browsers and email spam filters run before deciding whether to warn you about a link — just simplified and made visible, so you can see the reasoning instead of a black-box "this link is dangerous" popup.

**Important honesty note:** this tool only looks at the *shape* of the URL text. It doesn't check DNS records, doesn't look up blocklists, doesn't visit the page. A URL can score "LIKELY SAFE" and still be malicious, and a score can be inflated by a coincidence (e.g. a long but legitimate URL). The footer of the page says this explicitly — it's a teaching/triage tool, not a real security product.

---

## 2. The three layers of every web page

Every web page — this one included — is built from three languages stacked together:

| Layer | Language | Job | Where it lives in this file |
|---|---|---|---|
| **Structure** | HTML | What elements exist (buttons, text, boxes) | Between `<body>` and `</body>` |
| **Appearance** | CSS | Colors, spacing, fonts, animations | Inside `<style>...</style>` in the `<head>` |
| **Behavior** | JavaScript | What happens when you click, type, or load the page | Inside `<script>...</script>` tags near the bottom |

Because this is one `.html` file, all three are just glued together in sequence: head (with CSS) → body (with HTML) → scripts (with JS) at the end. In a bigger project these would usually be split into `.html`, `.css`, and `.js` files, but for a small app like this, one file keeps everything easy to find.

---

## 3. Reading order — how the browser processes this file

1. Browser reads `<head>` first → loads the Google Font, defines the favicon (the little tab icon), and reads all the CSS rules (but doesn't show anything yet).
2. Browser reads `<body>` → builds the visible page: the background canvas, the header, the input box, the report section (hidden at first), the footer.
3. Browser reaches the `<script>` tags at the bottom → runs the JavaScript. This is deliberately placed *after* the HTML so that when the JS tries to grab elements like `document.getElementById('urlInput')`, those elements already exist on the page. If scripts ran before the HTML existed, `getElementById` would return `null` and everything would break.
4. Nothing happens next until *you* do something — click SCAN, type Enter, click an example chip, move your mouse (for the background animation). JavaScript in a browser is **event-driven**: it sits idle and waits for events (clicks, key presses, page resize, mouse move) and only runs code in response.

---

## 4. CSS — how the visual design works

### 4.1 CSS variables (`:root`)
```css
:root {
  --bg: #0d1117;
  --panel: #141a24;
  --line: #232b38;
  --ink: #c9d1d9;
  --dim: #5b6572;
  --amber: #ffb020;
  --red: #ff4d4f;
  --green: #3fb950;
}
```
These are **custom properties** — named colors you define once and reuse everywhere with `var(--amber)` instead of retyping `#ffb020` fifty times. Change the value here once, and every element using `var(--amber)` updates automatically. This is why the severity colors (high=red, medium=amber, low=green) stay consistent across the whole app.

### 4.2 The reset (`* { box-sizing: border-box; margin: 0; padding: 0; }`)
Browsers apply their own default spacing to elements (headings have margin, lists have padding, etc.), and it's different browser to browser. This line wipes all of that to zero so you start from a predictable blank slate and add spacing back on purpose. `box-sizing: border-box` changes how width/height math works — padding and border get *included* in an element's stated width instead of added on top, which makes layout math much less confusing.

### 4.3 The animated background canvas
```css
#bgCanvas{
  position: fixed;
  inset: 0;
  z-index: -1;
}
```
- `position: fixed` pins it to the browser window (not the page — it won't scroll away).
- `inset: 0` is shorthand for `top:0; right:0; bottom:0; left:0` — it stretches to fill the whole screen.
- `z-index: -1` pushes it *behind* everything else. z-index controls stacking order — higher numbers sit on top, and this negative value shoves it under the default layer, behind your content.

The `<canvas>` element itself is just an empty rectangle — a "drawing surface." All the actual visuals (the wireframe globe, glowing embers, etc.) are drawn onto it by JavaScript, frame by frame. CSS only positions the canvas; it doesn't control what's drawn on it.

### 4.4 The gradient title text
```css
h1{
  background: linear-gradient(90deg, #00e5ff 0%, ... #ff2d2d 100%);
  -webkit-background-clip: text;
  background-clip: text;
  color: transparent;
}
```
This is a common CSS trick for gradient text: you paint a gradient as the element's *background*, then use `background-clip: text` to say "only show this background where the text characters are" and `color: transparent` so the actual text color doesn't cover it up. The result: letters that are filled with a color gradient instead of a flat color. The `-webkit-` prefixed version exists because some browsers (older Safari/Chrome) need the vendor-prefixed property name to support this.

### 4.5 Flexbox layouts
You'll see `display: flex` repeatedly (`.scanbar`, `.brand-row`, `.stat-row`, `.gauge-row`, `.finding`). Flexbox is a CSS layout mode for arranging items in a row or column that can grow, shrink, and align easily. Key properties used here:
- `align-items: center` — vertically centers items within the row
- `gap: 8px` — puts consistent spacing *between* items without needing manual margins
- `flex: 1` on `.scanbar input` — tells the input box to grow and fill all remaining space, so the SCAN button sits snugly at its natural width and the input takes everything else

### 4.6 Animations (`@keyframes`)
```css
@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: .35; }
}
```
`@keyframes` defines a named animation as a timeline: at 0% (the start) and 100% (the end) the opacity is fully visible, at 50% (the midpoint) it fades to 35%. Then an element uses it with `animation: pulse 1.8s infinite;` — play the `pulse` keyframes over 1.8 seconds, forever. This is what makes the little status dots (the eyebrow bullet, the stat-chip dots) breathe in and out.

Similarly `@keyframes logoFloat` moves and slightly rotates the shield logo up and down, and `@keyframes rise` makes each finding row fade in and slide up when the report appears (`animation: rise .35s ease forwards` on `.finding`, with a staggered `animation-delay` set in JavaScript so they cascade in one after another instead of all popping in at once).

### 4.7 Responsive-ish sizing
`.wrap { max-width: 1100px; width: 95%; margin: 40px auto; }` — the content area is *at most* 1100px wide, but on smaller screens shrinks to 95% of the viewport width. `margin: 40px auto` centers it horizontally (auto margins on left/right split the remaining space evenly) with 40px of breathing room top and bottom.

---

## 5. HTML structure — what's actually on the page

Simplified skeleton (some attributes omitted for clarity):

```
<body>
  <canvas id="bgCanvas"></canvas>       <!-- animated background, drawn by JS -->

  <div class="wrap">                    <!-- centers + constrains all content -->
    <header>
      <div class="eyebrow">...</div>    <!-- small uppercase label -->
      <div class="brand-row">
        <svg class="logo-mark">...</svg> <!-- shield icon -->
        <h1>URL Threat Scanner</h1>
      </div>
      <p class="sub">...</p>            <!-- description paragraph -->
      <div class="stat-row">...</div>   <!-- credibility badges -->
    </header>

    <div class="scanbar">
      <input id="urlInput" ...>         <!-- where you type the URL -->
      <button id="scanBtn" onclick="runScan()">SCAN</button>
    </div>

    <div id="scanline-wrap">            <!-- thin animated bar shown while "scanning" -->
      <div class="scanline"></div>
    </div>

    <div class="examples">...</div>     <!-- clickable example URLs -->
    <div class="history-row" id="historyRow">...</div>  <!-- recent scans -->

    <div id="report">                   <!-- hidden until first scan -->
      <div class="gauge-row">
        <div class="gauge">...</div>    <!-- circular score meter -->
        <div class="verdict-block">...</div>
        <button id="copyBtn">COPY</button>
      </div>
      <div class="findings" id="findings"></div>  <!-- filled in by JS -->
    </div>

    <footer>...</footer>
  </div>

  <script>...</script>  <!-- analysis engine + UI logic -->
  <script>...</script>  <!-- background animation -->
</body>
```

A few HTML details worth understanding:

- **`id` vs `class`**: `id="urlInput"` is a *unique* name used exactly once, meant for JavaScript to grab that one specific element (`document.getElementById('urlInput')`). `class="ex-chip"` is meant to be reused on many elements at once, mainly for CSS styling (and sometimes for JS to grab a whole group with `querySelectorAll`).
- **Inline `onclick`**: `<button onclick="runScan()">` runs the `runScan()` JavaScript function the moment the button is clicked. It's a quick, readable way to wire up simple interactions directly in the HTML, though larger apps often prefer attaching listeners in JavaScript instead (which this file also does in one place — see §7.1).
- **The `<svg>` logo**: SVG (Scalable Vector Graphics) is XML-based image code you can write by hand — shapes described as math (paths, circles) rather than pixels, so it scales to any size with no blur. The shield shape is a single `<path>` with a `d` attribute containing draw commands (`M` = move to, `L` = line to, `C` = curve to), and it's stroked with a `<linearGradient>` so the outline itself shows the blue-to-red gradient.

---

## 6. The analysis engine — the actual "brain" of the app

This lives in the first `<script>` block. It's pure logic — no DOM manipulation, no visuals. In theory you could copy this whole chunk into a Node.js file and it would still work, which is a good sign of clean separation between "logic" and "UI."

### 6.1 Reference data (the "known facts" the checks compare against)
```js
const KNOWN_BRANDS = ["google", "paypal", "microsoft", ...];
const SUSPICIOUS_TLDS = [".zip", ".mov", ".xyz", ".top", ...];
const URL_SHORTENERS = ["bit.ly", "tinyurl.com", ...];
const SUSPICIOUS_KEYWORDS = ["login", "signin", "verify", ...];
```
These are just arrays (lists) of strings. They're the "knowledge" the scanner uses — a real product would pull these from a constantly-updated threat-intel feed; here they're hardcoded for simplicity and transparency (you can see exactly what it's checking against, nothing hidden).

### 6.2 `levenshtein(a, b)` — measuring "how different are two words?"
This is a classic algorithm called **edit distance**: the minimum number of single-character insertions, deletions, or substitutions needed to turn string `a` into string `b`. For example, `levenshtein("paypa1", "paypal")` returns `1`, because you only need to change one character (`1` → `l`) to match.

Why it matters here: typosquatting domains (fake lookalikes like `paypa1.com` or `gooogle.com`) are usually *one or two characters off* from the real brand name. Edit distance is exactly how you catch "this is suspiciously close to a real word" programmatically, instead of needing an exact-match list of every possible misspelling (which would be impossible to maintain).

How the algorithm works internally: it builds a table (here flattened into a rolling `prev`/`cur` array to save memory) where each cell represents "the edit distance between the first *i* characters of `a` and the first *j* characters of `b`." It fills the table left-to-right, top-to-bottom, and each cell's value depends only on the three neighboring cells already computed (left, above, diagonal). This "build up from small subproblems" pattern is called **dynamic programming** — you'll see it constantly in algorithms once you know to look for it.

### 6.3 `entropy(s)` — measuring "how random does this text look?"
This is **Shannon entropy**, borrowed from information theory. In plain terms: it measures how *unpredictable* the characters in a string are. `"aaaaaaaa"` has very low entropy (totally predictable, one repeated character). `"x7q2plk9"` has high entropy (looks random).

Why it matters: phishing infrastructure is often auto-generated by scripts — subdomains like `x7f9k2.evil-domain.com` — and auto-generated strings tend to look statistically more "random" than real words, because real words follow the predictable letter patterns of a human language. High entropy on a domain label is a soft signal of "this might be machine-generated," which is exactly the kind of thing bots use to dodge static blocklists.

How it's calculated: for each unique character, count how often it appears, turn that count into a probability (frequency ÷ total length), then sum up `-p * log2(p)` for every character. This is literally the formula used in classical information theory to measure "how many bits of information do I need, on average, to describe this data."

### 6.4 `analyze(rawUrl)` — the main function, step by step

This function takes the raw text you typed and returns either an error or a full result object. Here's the flow:

**Step 0 — Normalize and parse**
```js
let url = rawUrl.trim();
if (!/^\w+:\/\//.test(url)) url = "http://" + url;
let parsed = new URL(url);
```
- `.trim()` removes accidental leading/trailing spaces.
- The regex `/^\w+:\/\//` checks "does this already start with something like `http://` or `ftp://`?" If someone just types `wikipedia.org` with no protocol, this prepends `http://` so the browser's built-in `URL` parser can understand it.
- `new URL(url)` is a built-in browser API that breaks a URL string into its meaningful parts (`protocol`, `hostname`, `pathname`, `search`, `port`, etc.) — no manual string-splitting needed. If the text genuinely can't be parsed as a URL at all, this throws an error, which is caught and turned into a friendly `{ error: "..." }` result instead of crashing the page.

**The `add()` helper**
```js
const findings = [];
const add = (points, reason, sev) => findings.push({ points, reason, sev });
```
This is a small local function (a "closure" — it's defined inside `analyze()` and can see `analyze()`'s local `findings` array) that every check below calls to record a red flag: how many points it's worth, a human-readable explanation, and a severity label (`"high"`, `"medium"`, `"low"`).

**Steps 1–9 — the individual checks**, each independent and each calling `add()` if triggered:

| Step | What it checks | Why it's a red flag |
|---|---|---|
| 1. Raw IP | Is the hostname literally an IP address (`192.168.1.1`) instead of a domain name? | Legitimate sites almost always use a named domain; a bare IP hides the real owner and skips normal domain registration scrutiny |
| 2. HTTPS | Is the protocol `http:` instead of `https:`? | No encryption — data (including anything you type) travels in plain text |
| 3. `@` symbol | Does the URL contain an `@`? | Browsers historically let you write `http://real-site.com@evil.com` — everything before `@` is ignored as fake "credentials," and the browser actually visits `evil.com`. A classic obfuscation trick |
| 4. Length | Is the URL over 75 characters? | Very long URLs are often used to bury the real destination or cram in extra tracking/obfuscation parameters |
| 5. Hyphens & subdomains | 3+ hyphens in the domain, or a hyphen combined with a brand name, or 3+ subdomain levels | Real brands rarely register hyphen-heavy domains; scammers use them to construct fake-looking-legit names (`secure-paypal-login-verify.com`) |
| 6. Typosquatting | Uses `levenshtein()` to compare domain chunks against `KNOWN_BRANDS` | Catches near-miss spellings of real brands — the highest-weighted check (30 points) because it's a strong signal |
| 7. Entropy | Uses `entropy()` on the first label; flags if entropy > 3.8 AND length > 8 | Flags likely auto-generated subdomains (see §6.3) |
| 8. Shorteners / TLDs / keywords | Is the host a known shortener? Does it end in a suspicious TLD? Does it contain scam keywords like "verify" or "urgent"? | Shorteners hide the real destination; certain TLDs are cheap and heavily abused; urgency/credential language is classic social-engineering bait |
| 9. Punycode / port | Does the hostname start with `xn--` (punycode encoding)? Is a non-standard port specified? | Punycode can be used for **homograph attacks** — encoding lookalike Unicode characters (e.g. a Cyrillic "а" that looks identical to a Latin "a") so a domain *looks* like `apple.com` to the human eye but is actually a completely different registered domain. Odd ports can indicate non-standard, possibly malicious hosting setups |

Notice the checks are ordered from "most structurally certain" (raw IP, `@` symbol) toward "softer statistical signals" (entropy) — a reasonable way to organize a rule list, most-confident first.

**Step 10 — Score & verdict**
```js
const score = Math.min(100, findings.reduce((s, f) => s + f.points, 0));
```
`.reduce()` walks through the `findings` array and sums up every `.points` value into one running total (`s` starts at nothing implicit here — actually starts at the first element pattern, accumulating `s + f.points` for each finding), then `Math.min(100, ...)` caps it so the score never exceeds 100 even if many checks fire at once.

Then a simple if/else ladder converts the numeric score into a human verdict and a color:
```js
if (score >= 70)      → "HIGH RISK"        → red
else if (score >= 40) → "SUSPICIOUS"       → amber
else if (score >= 15) → "LOW RISK"         → amber
else                  → "LIKELY SAFE"      → green
```

**Step 11 — Return the result**
```js
return {
  score, verdict, verdictColor,
  findings: findings.sort((a, b) => b.points - a.points),
  displayUrl: url
};
```
The findings are sorted highest-points-first so the most serious red flags appear at the top of the report. This whole object is what gets handed off to the "UI wiring" section next, which is a clean boundary: the analysis engine doesn't know or care about HTML — it just computes and returns data.

---

## 7. UI wiring — connecting logic to the screen

This is the second half of the first `<script>` block. Its job: react to user actions, call `analyze()`, and update the DOM (the live, in-browser representation of your HTML) to show the results.

### 7.1 `runScan()`
```js
function runScan() {
  const input = document.getElementById('urlInput').value;
  if (!input.trim()) return;
  ...
  btn.disabled = true; btn.textContent = "SCANNING";
  scanWrap.classList.add('active');

  setTimeout(() => {
    const result = analyze(input);
    ...
    renderReport(result);
    addToHistory(result);
  }, 650);
}
```
- `document.getElementById('urlInput').value` reads whatever text is currently in the input box.
- If it's empty (after trimming whitespace), the function just `return`s early — no point scanning nothing.
- The button is disabled and its text changed to "SCANNING" so you get visual feedback that something's happening, and can't double-click to trigger overlapping scans.
- `classList.add('active')` turns on the little animated scan-line bar (a CSS class toggle — the `.active` styles are what actually give it height and show the sweep animation, defined back in the CSS).
- `setTimeout(..., 650)` deliberately **delays** running the actual analysis by 650 milliseconds. The analysis itself is instant (it's just string checks), but a 0ms "scan" would feel fake — the artificial delay sells the illusion of the tool actually doing work, similar to why real scanning/loading UIs often have a minimum display time even when the underlying operation is fast.
- Inside the delayed callback: run `analyze()`, reset the button, hide the scan-line, then hand the result to `renderReport()` and `addToHistory()`.

### 7.2 `renderReport(result)`
Takes the result object and paints it onto the page:
- Sets `#report`'s `display` to `'block'` (it starts as `display: none` in CSS, hidden until the first scan).
- Fills in the URL text, the numeric score, and the verdict text/color.
- Animates the circular gauge: it's an SVG `<circle>` using `stroke-dasharray` (total length of the dashed outline, fixed at `264` — roughly the circle's circumference) and `stroke-dashoffset` (how much of that dash pattern to *hide*, essentially "how much of the circle is not drawn yet"). Setting `offset = circumference - (circumference * score / 100)` means a higher score reveals more of the ring. `requestAnimationFrame` schedules the actual style change for the next screen repaint, which lets the CSS `transition` on `stroke-dashoffset` (defined inline: `transition: stroke-dashoffset .8s ease`) animate smoothly from empty to filled instead of jumping instantly.
- Clears and rebuilds the findings list: for each finding, creates a `<div>`, sets its inner HTML to a badge + text + point value, and gives it a staggered `animation-delay` (`i * 0.05` seconds) so each row fades in slightly after the previous one — a cheap way to make a static list feel alive.

### 7.3 History feature (`addToHistory`, `renderHistory`, `severityBucket`)
- `scanHistory` is a plain JavaScript array kept in memory (it resets if you reload the page — there's no saving to disk or a database here).
- `addToHistory()` removes any earlier entry for the same URL (so re-scanning something doesn't create duplicates), puts the new result at the front (`unshift`), and trims the array to a max of 6 entries.
- `renderHistory()` rebuilds the little chip row: for each history entry, it figures out a severity bucket (high/medium/low) via `severityBucket(score)`, creates a clickable button styled with that severity's color, and wires its `onclick` to re-fill the input and re-run the scan — so clicking history is just a shortcut back to `runScan()`.

### 7.4 `copyReport()`
Builds a plain-text summary (score, verdict, every finding) as an array of lines, joins them with newlines, and uses the browser's `navigator.clipboard.writeText()` API to copy it. This API is **asynchronous** (it returns a Promise, hence `.then()`/`.catch()`) because clipboard access can require a moment and can fail (e.g. if the browser blocks it in some contexts) — the `.then()` branch flips the button text to "COPIED" briefly and reverts it after 1.5 seconds with `setTimeout`; the `.catch()` branch handles failure gracefully instead of leaving you guessing.

### 7.5 `loadExample(u)`
Just fills the input with a preset URL string and calls `runScan()` — this is how the example chips and any future "quick test" buttons work.

### 7.6 The one addEventListener
```js
document.getElementById('urlInput').addEventListener('keydown', e => {
  if (e.key === 'Enter') runScan();
});
```
Unlike the buttons (which use inline `onclick`), this listens for keyboard events on the input box specifically, checking if the pressed key was `Enter`, so you can scan by pressing Enter instead of only by clicking the button. This is attached via `addEventListener` rather than an inline HTML attribute because keyboard events on a specific element are a more natural fit for JS-side wiring — there's no equivalent simple inline HTML attribute for "on Enter key."

---

## 8. The background animation — a from-scratch 3D engine

This lives in the second `<script>` block, wrapped in an **IIFE** (Immediately Invoked Function Expression):
```js
(function () { ... })();
```
This is a common pattern to create a private scope — everything declared inside (`canvas`, `rings`, `embers`, etc.) stays invisible to the rest of the page's code, avoiding naming collisions with the first script block. It runs itself immediately once, on page load.

There's no 3D library here (no Three.js, no WebGL) — it's all 2D canvas drawing combined with hand-written 3D math. This is worth understanding because it demystifies what a "3D engine" actually does under the hood at a basic level.

### 8.1 Representing points in 3D space
Every point (a globe vertex, an ember) is just an object `{x, y, z}` — three numbers describing a position in 3D space, with the origin `(0,0,0)` at the center of the scene.

### 8.2 Building a wireframe sphere
```js
const phi = (i / LAT_STEPS) * Math.PI;
const ringR = GLOBE_R * Math.sin(phi);
const y = GLOBE_R * Math.cos(phi);
```
This is **spherical coordinates** math: instead of picking `x, y, z` directly, you pick an angle (`phi`, latitude-like) and compute where that angle lands on a sphere of radius `GLOBE_R` using sine and cosine. The code generates a series of horizontal rings (lines of latitude) and vertical rings (lines of longitude), each made of many small connected points, which together look like a wireframe globe — the same idea as the grid lines on a world map globe.

### 8.3 Projecting 3D onto a 2D screen (`project()`)
A screen only has two dimensions (x, y pixels), so 3D points need to be **projected** down to 2D before they can be drawn. The `project()` function does two things:

1. **Rotation** — rotates the point around the Y axis (left-right spin) and X axis (up-down tilt) using standard 2D rotation matrix math (the `cos`/`sin` formulas), based on the current `rotY`/`rotX` angles.
2. **Perspective divide** — `const scale = FOV / depth;` — this is the core trick of perspective: things farther away (`depth` larger) get a smaller `scale`, so they appear smaller and closer to the center, exactly like how real-world perspective works (distant objects look smaller). `sx`/`sy` are the final on-screen pixel coordinates after applying that scale.

### 8.4 Mouse-driven rotation
```js
window.addEventListener('mousemove', (e) => {
  targetRotY = mouseX * 0.7;
  targetRotX = mouseY * 0.4;
});
...
rotY += (targetRotY - rotY) * 0.02 + 0.0016;
```
Mouse position is converted into a *target* rotation angle. But instead of snapping straight to that angle, each frame nudges `rotY` a small fraction (`* 0.02`) of the way toward the target. This is a common **easing/lerp** (linear interpolation) technique — it makes the rotation feel smooth and slightly delayed/springy rather than jerky and instant. The added `+ 0.0016` keeps a small constant rotation going even when the mouse isn't moving, so the globe never looks frozen.

### 8.5 The render loop (`tick()`)
```js
function tick() {
  ... draw everything ...
  requestAnimationFrame(tick);
}
tick();
```
This is the animation heartbeat. `requestAnimationFrame` asks the browser "call this function again right before the next screen repaint" — typically 60 times per second. Each call to `tick()` clears the canvas and redraws everything from scratch at the current rotation/positions, which is how you get smooth motion: it's really a rapid slideshow of still frames, same as film. Each `tick()`:
1. Clears the canvas and paints a dark radial-gradient base + a "flickering" vignette (small random jitter added to opacity each frame for a subtle unstable-power feel).
2. Draws a glowing core circle at screen center, pulsing via `Math.sin(t * 0.05)` — sine waves are the standard way to create smooth back-and-forth oscillation over time.
3. Draws all the globe wireframe rings, with each ring's opacity based on its average depth (`avgZ`) — rings further away/behind fade out, reinforcing the 3D depth illusion.
4. Updates and draws the drifting embers: each has its own slow random velocity (`drift`), bounces back when it strays too far or gets too close to the globe, and pulses in brightness. They're sorted by depth (`sort((a,b) => a.p.z - b.p.z)`) before drawing — this is the **painter's algorithm**, drawing far-away things first so closer things naturally get drawn on top and occlude them, mimicking how a painter builds a scene back-to-front.
5. Draws faint horizontal scanlines across the whole canvas (a subtle repeating semi-transparent fill every 3 pixels) for a retro-CRT/surveillance-monitor texture.
6. Occasionally (randomly, roughly every few seconds) triggers a "glitch tear": grabs a thin horizontal strip of already-drawn pixels (`getImageData`), and redraws it shifted sideways (`putImageData` with an x-offset) — a cheap but convincing digital-glitch effect.

---

## 9. Key programming concepts this project teaches, if you want to study further

- **Regular expressions (regex)** — used throughout for pattern matching (`/^\w+:\/\//`, `/^(\d{1,3}\.){3}\d{1,3}$/` for IP detection). Worth learning regex syntax properly if these look cryptic; it pays off fast.
- **Array methods**: `.filter()`, `.map()`, `.reduce()`, `.sort()`, `.forEach()`, `.some()` — functional-style ways to transform lists without manual `for` loops. Every one of these appears in this file at least once.
- **Closures** — functions (like `add()` inside `analyze()`) that "remember" variables from the scope they were defined in, even after that outer function keeps running.
- **The DOM (Document Object Model)** — the live tree of elements the browser builds from your HTML, which JavaScript can read and mutate (`getElementById`, `.textContent`, `.style`, `.classList`, `.innerHTML`, `.appendChild`).
- **Events and event-driven programming** — code that reacts to things happening (clicks, key presses, mouse movement, resize) rather than running top-to-bottom once.
- **Promises and async APIs** — `navigator.clipboard.writeText()` returning a `.then()/.catch()`-able Promise is a small taste of asynchronous JavaScript, where an operation's result isn't available immediately.
- **CSS custom properties, flexbox, keyframe animations, and gradient-clipped text** — modern CSS techniques for maintainable, animated styling without JavaScript.
- **Canvas 2D API** — direct pixel/shape drawing (`fillRect`, `arc`, `createRadialGradient`, `getImageData`/`putImageData`) as the foundation beneath any "no-library" browser graphics.
- **Basic trigonometry and 3D math** — sine/cosine for rotation and orbiting motion, spherical coordinates for generating a sphere, and perspective-divide for faking 3D depth on a flat screen.
- **Algorithms**: Levenshtein edit distance (dynamic programming) and Shannon entropy (information theory) — both genuinely used in real-world security and NLP tooling, not just toy examples.

---

## 10. Known limitations (good to understand, not bugs to "fix" blindly)

- **No live network checks** — by design, for privacy and simplicity, and because browser CORS restrictions would block fetching arbitrary sites anyway from client-side JS.
- **History is memory-only** — refreshing the page clears it; there's no `localStorage` or backend database.
- **The brand/keyword/TLD lists are static and small** — a production tool would need these constantly updated from a real threat feed.
- **Scoring weights are heuristic**, not derived from real-world data — they're reasonable estimates of severity, not a calibrated statistical model.

---

*If you're studying this to learn web development: read section order 5 → 4 → 6 → 7 → 8. Structure first, then how it looks, then how it thinks, then how it responds to you, then the flashy extra. That's also roughly the order a browser processes it.*
