---
name: site-clone
description: >
  Clone any public website into a faithful HTML/CSS replica using browser scraping,
  CSS extraction, and visual analysis. Triggered by "copy website X", "buat clone X",
  "replika website".
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux]
tags: [site-clone, web-replica, reverse-engineering, html, css, scrape, deploy]
triggers:
  - "copy website"
  - "clone site"  
  - "replikasi website"
  - "buat seperti"
  - "mirip kayak"
  - "copy dari URL"
related_skills: [claude-design, popular-web-designs, sketch]
---

# Site Clone

Use when the user wants to **replicate an existing public website's design** as an HTML/CSS artifact, then deploy it.

## When to Use This Skill vs `popular-web-designs` vs `claude-design`

| Skill | Trigger | What It Does |
|---|---|---|
| **site-clone** | User gives a URL or says "copy website X" | Scrapes the target → extracts CSS/fonts/layout → rebuilds faithful replica |
| **popular-web-designs** | "make it look like Stripe/Linear" (known brand from catalog) | Provides pre-baked design tokens for 54 famous sites |
| **claude-design** | "design something original" / "better looking" | Design process, composition rules, anti-slop checklist |

**Rule:** URL = `site-clone`. Brand name from popular list = `popular-web-designs`. Original design = `claude-design`.

## Workflow

### 1. Scrape & Extract

Write a Playwright scraper script in `/home/ubuntu/.hermes/skills/web3/seed-phrase-recovery/scripts/` or temp `/tmp/`:

```js
const { chromium } = require("/home/ubuntu/wallet/node_modules/playwright-core");
(async () => {
  const browser = await chromium.launch({ 
    headless: true,
    executablePath: "/home/ubuntu/.cache/ms-playwright/chromium_headless_shell-1234/chrome-headless-shell-linux64/chrome-headless-shell"
  });
  const page = await browser.newPage();
  
  // Load, scroll, re-load for lazy content
  await page.goto(url, { waitUntil: "domcontentloaded" });
  await page.waitForTimeout(5000);
  await page.evaluate(async () => {
    for (let i = 0; i < document.body.scrollHeight; i += 500) {
      window.scrollTo(0, i);
      await new Promise(r => setTimeout(r, 100));
    }
  });
  await page.waitForTimeout(3000);
  await page.goto(url, { waitUntil: "networkidle" });
  await page.waitForTimeout(5000);
  
  // Key extractions
  const bodyText = await page.evaluate(() => document.body.innerText);
  const links = await page.evaluate(() => [...document.querySelectorAll("a")].map(a => ({href:a.href, text:a.textContent.trim()})));
  const imgs = await page.evaluate(() => [...document.querySelectorAll("img")].map(i => ({src:i.src, alt:i.alt})));
  const classes = await page.evaluate(() => { /* collect all class names */ });
  const meta = await page.evaluate(() => { /* OG tags */ });
  const sections = await page.evaluate(() => { /* section elements with text/classes */ });
  
  // Screenshots at breakpoints
  for (const w of [1920, 1280, 768]) {
    await page.setViewportSize({ width: w, height: 1080 });
    await page.reload({ waitUntil: 'networkidle' });
    await page.waitForTimeout(2000);
    await page.screenshot({ path: `/tmp/${sitename}_${w}px.png`, fullPage: true });
  }
  await browser.close();
})();
```

Run background → parse output → save to `/tmp/<sitename>_scrape.txt`.

Also fetch external CSS files directly (they're often not in AX tree for Next.js sites):
```js
const r = await fetch("https://site.com/_next/static/chunks/main.css");
const buf = Buffer.from(await r.arrayBuffer());
// ~50KB of Tailwind-generated CSS with all resolved values
```

### 2. Analyze Extracted Data

Extract from scraped data:
- **Colors** from CSS custom properties (`--arc-blue`, `--void`, etc.) or Tailwind color values
- **Fonts** from `@font-face` declarations → map to Google Fonts equivalents
- **Sections** from headings + body text classification
- **Components** from `.btn-*` / `.card-*` class definitions
- **Effects** from `@keyframes`, gradients, filter declarations
- **Layout** from CSS grid/flex patterns in the generated CSS

### 3. Build Replica

Template pattern: single self-contained HTML file
- Inline `<style>` with extracted CSS custom properties
- Google Fonts CDN for font replacement
- Sections matching original layout structure
- Canvas-generated procedural art as placeholder for missing images
- Scroll reveal + animation JS

Save to: `/home/ubuntu/nft-work/{project-name}/index.html`

### 4. Deploy

```bash
cd /home/ubuntu/nft-work/{project-name}
npx vercel link --yes
npx vercel --prod --confirm
```

Get live URL from `vercel ls` output.

### 5. Verify

Use `browser_navigate` + `browser_vision` on the deployed URL. Or screenshot via headless Chromium + `vision_analyze` to compare against original screenshots.

## VPS Playwright Notes

- `playwright-core` required from `/home/ubuntu/wallet/node_modules/playwright-core` (not global)
- Executable: `/home/ubuntu/.cache/ms-playwright/chromium_headless_shell-1234/chrome-headless-shell-linux64/chrome-headless-shell`
- System deps already installed (`npx playwright install-deps chromium`)
- After new installs: run `npx playwright install chromium` to update binaries

## Common Dark Sci-Fi / Cyberpunk Effect Recipes

| Effect | CSS Pattern |
|---|---|
| Grid background | `linear-gradient(var(--line) 1px, transparent 1px)` x2 rotated 90deg + radial mask fade |
| CRT scanlines | `repeating-linear-gradient(#0000 0 3px, #02081a38 4px)` fixed overlay |
| Film grain | SVG noise filter animated position with `steps()` keyframes |
| Neon glow | `box-shadow: 0 0 24px -4px rgba(46,107,255,0.9)` |
| Marquee ticker | `animation: marquee-x 25s linear infinite` with duplicated content |
| Button shine sweep | Pseudo-element with gradient, translateX(-120%)→translateX(120%) on hover |
| Ring pulse | Scale(0.8→1.7) + opacity 0.6→0 |
| CRT boot animation | Multi-step brightness/saturation shifts fading to normal |

See `references/arcadians-study.md` for detailed color palette, typography mapping, and section breakdown.

## Pitfalls

1. **Next.js CSS not in AX tree** — fetched files must be grabbed via direct HTTP `fetch()`, not DOM selectors.
2. **Font loading race in headless** — Google Fonts may not load before screenshot. Add `page.waitForTimeout(3000)` after fonts load or use `page.addStyleTag()`.
3. **Full-page dark theme screenshots can be blank** — renderer doesn't apply CSS on some headless Chromium configs. Use viewport screenshots first.
4. **Image CDN URLs expire** — Vercel `/_next/image?url=...&dpl=...` has session tokens. Don't hardcode these; download actual assets via direct URLs instead.
5. **JS-heavy SPAs** — client-rendered content won't appear in `page.content()`. Must use `await page.evaluate(() => document.body.innerText)`.
6. **Tailwind v4 `@property` syntax** — newer Tailwind versions use CSS `@property` for typed custom properties. These appear in generated CSS but may confuse regex parsers expecting simple variables.
7. **Responsive layouts vary drastically per breakpoint** — always screenshot at 1920px AND 768px minimum. Mobile styles may hide/show completely different elements.
8. **Don't over-copy copyrighted UI** — replicate layout patterns and visual *feel*, not exact proprietary components or copied asset content.
