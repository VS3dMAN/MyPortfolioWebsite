# Performance Optimization Plan — Portfolio Site

Written 2026-07-31. Target: `index.html`, `refresh-fragrances.html`, `premolc.html`.
Every item is free, needs no build tooling, and preserves the current design exactly.

---

## Implementation status (updated 2026-07-31)

**Done:**
- P1 — fonts self-hosted, subsetted to latin + the punctuation actually used, `font-display: swap`,
  preload on Syne 800 + Space Grotesk 400. Google Fonts origin removed entirely from all 3 pages.
  228 KB of font files → 110 KB, and zero third-party requests remain.
- P2 — small carousel variants (`-sm.webp`) generated for `refresh/` and `premolc/`; homepage now
  loads those instead of gallery-sized files. Hero portrait re-encoded 900×1200 → 800×1067.
- P2C — `content-visibility: auto` + `contain-intrinsic-size` on `.shot` in both gallery pages.
- P3 — nav scroll handler throttled to one style write per frame and made passive; carousel drift
  loop now cancelled when off-screen instead of early-returning every frame.
- P5 — `_headers` with immutable caching for `/images/*` and `/fonts/*`; hover-prefetch on subpage
  links.
- P6 — `<noscript>` reveal guard on all 3 pages; `sitemap.xml` and `robots.txt` added.

**Not done — needs the live host, not the code:**
- P4 — Brotli/gzip. Automatic on Netlify, Vercel and Cloudflare Pages; verify after deploy with
  `curl -sI -H "Accept-Encoding: br" <url> | grep -i content-encoding`.
- P6.2 — per-section `contain-intrinsic-size` values still use the blanket 800 px. Measure real
  section heights in DevTools once deployed and replace.

---

## Current state

| | |
|---|---|
| Pages | 3 static HTML files, no framework, no bundler |
| `index.html` | ~72 KB (inline CSS + JS + SVG sprite + JSON-LD) |
| Images | 45 WebP files, ~1.3 MB total |
| Blocking requests | 1 — the Google Fonts stylesheet |
| JS | ~7 KB inline, no dependencies |

The site is already fast. The remaining wins are (a) the font chain, (b) image bytes actually
delivered vs. displayed, and (c) main-thread work during scroll.

---

## Priority 1 — Kill the render-blocking font request

**Problem.** `<link href="fonts.googleapis.com/css2?...">` is render-blocking. The browser must
resolve DNS → connect → TLS → fetch the CSS → *then* discover and fetch 3 font files from a second
origin (`fonts.gstatic.com`). On a cold 4G connection that is 400–900 ms before any text paints.
Three families (Syne, Space Grotesk, JetBrains Mono) at multiple weights is 6–10 font files.

**Fix — self-host the fonts.** This removes an entire origin from the critical path.

1. Download the WOFF2 files (google-webfonts-helper, or fetch the `css2` URL with a modern
   User-Agent and pull the `.woff2` URLs out of the response).
2. Keep only the weights actually used. Audit first — the current link requests
   Syne 400/500/600/700/800, Space Grotesk 400/500/600/700, JetBrains Mono 400/500. Grep the CSS
   for `font-weight` and drop every weight nothing references. Expect to cut this roughly in half.
3. Save to `fonts/` and declare them locally:

   ```css
   @font-face {
     font-family: 'Syne';
     src: url('fonts/syne-700.woff2') format('woff2');
     font-weight: 700;
     font-display: swap;   /* text paints immediately in fallback, swaps when ready */
     unicode-range: U+0000-00FF, U+2010-2027; /* latin subset only */
   }
   ```
4. Preload only the two fonts used above the fold (Syne 700/800 for the hero name, Space Grotesk 400
   for body):
   ```html
   <link rel="preload" href="fonts/syne-800.woff2" as="font" type="font/woff2" crossorigin>
   ```
5. Delete the `<link rel="preconnect">` tags for `fonts.googleapis.com` / `fonts.gstatic.com` —
   dead weight once nothing loads from those origins.

**Expected gain:** 300–800 ms off First Contentful Paint on cold mobile. This is the single
largest win available.

**Bonus:** also removes a third-party request, which is a small privacy/GDPR improvement.

---

## Priority 2 — Stop shipping oversized images

**Problem.** Several images are served far larger than they render.

| File | Intrinsic | Rendered (desktop) | Waste |
|---|---|---|---|
| `portrait.webp` | 900×1200 | 380×460 | ~2.4× oversampled |
| project visuals | 1280×720 | ~506×280 | ~2.5× |
| `premolc/*.webp` | 760×570 | 150px tall in carousel | ~4× |
| `refresh/*.webp` | ~720 | 240px tall in carousel | ~3× |

At 2× DPR the carousel images still only need ~480px tall. Everything above that is wasted bytes.

**Fix A — responsive sources.** For the images that appear at two very different sizes (carousel
thumbnail on `index.html`, full size on the gallery pages), generate a small variant and use
`srcset` so each context downloads what it needs:

```html
<img src="images/premolc/p1-480.webp"
     srcset="images/premolc/p1-480.webp 480w, images/premolc/p1.webp 760w"
     sizes="(max-width: 900px) 40vw, 200px"
     width="760" height="570" loading="lazy" decoding="async" alt="...">
```

Generate the variants with the Pillow script already used in this project — one pass, no tooling to
install.

**Fix B — re-encode at the right quality.** Current WebP quality is 82. For images that render at
150–280 px, quality 72–75 is visually identical and 25–35% smaller. Re-encode the carousel sets
(`refresh/`, `premolc/`) at the smaller dimensions and lower quality.

**Fix C — the gallery pages are the real hot spot.** `refresh-fragrances.html` loads 27 images and
`premolc.html` loads 12. They are `loading="lazy"`, but the masonry `columns` layout means the
browser can't know final positions until images decode, so lazy-loading fires more eagerly than you
want. Add `content-visibility: auto` + `contain-intrinsic-size` to `.shot` so off-screen figures
skip layout and paint entirely.

**Expected gain:** 40–60% reduction in image bytes on `index.html`; larger on the gallery pages.

---

## Priority 3 — Reduce main-thread work during scroll

**Problem areas in the current JS:**

1. **Two `rAF` loops always running.** `setupCarousel` starts a `requestAnimationFrame` drift loop
   per carousel — two loops that call `requestAnimationFrame` on *every* frame regardless of whether
   they do anything, and early-return inside. They are cheap, but they keep the main thread from
   ever going fully idle, which costs battery on phones.

   **Fix:** cancel the loop when `inView` goes false and restart it in the IntersectionObserver
   callback, instead of running a no-op every frame.

2. **The nav scroll handler is unthrottled.** `initNav` runs a `scroll` listener that reads
   `window.scrollY` and calls `classList.toggle` on every scroll event — that's a style write on a
   listener that can fire 100+ times/second.

   **Fix:** wrap in a `requestAnimationFrame` guard (the same `queued` pattern already used in
   `initReveal`'s sweep), and mark the listener `{ passive: true }`.

3. **Parallax + cursor glow** already gated behind `(hover: hover) and (pointer: fine)`, so phones
   skip them. Good — no change needed.

4. **Carousel track duplication** doubles the DOM node count for 30 images. Acceptable, but combine
   with Priority 2's smaller variants so the doubled nodes are cheap.

**Expected gain:** smoother scroll on mid-range Android, measurably lower Interaction to Next Paint.

---

## Priority 4 — Reduce transferred HTML

**Problem.** `index.html` is ~72 KB uncompressed and inlines everything.

**Fix — enable compression at the host.** This is a config change, not a code change, and it is the
cheapest win on this list. HTML/CSS/JS/SVG compress ~75–80%; 72 KB becomes ~16 KB.

- **Netlify / Vercel / Cloudflare Pages:** Brotli is on by default. Confirm with
  `curl -sI -H "Accept-Encoding: br" <url> | grep -i content-encoding`.
- **GitHub Pages:** gzip only, still fine.
- Do **not** compress the WebP files — already compressed, recompression wastes CPU for ~0 gain.

**Do not split the inline CSS/JS into separate files.** For a site this size, inlining is correct:
one request beats three, and the CSS is needed for first paint anyway.

---

## Priority 5 — Caching and delivery

1. **Cache headers.** Images and fonts never change once published; HTML does.
   ```
   /images/*   Cache-Control: public, max-age=31536000, immutable
   /fonts/*    Cache-Control: public, max-age=31536000, immutable
   /*.html     Cache-Control: public, max-age=0, must-revalidate
   ```
   On Netlify this goes in `_headers`; on Vercel, `vercel.json`. Both free.

2. **Prefetch the gallery pages on hover.** When a visitor hovers "See All", the subpage is almost
   certainly the next navigation. Cheap, and makes it feel instant:
   ```js
   document.querySelectorAll('a[href$=".html"]').forEach(a => {
     a.addEventListener('mouseenter', () => {
       const l = document.createElement('link');
       l.rel = 'prefetch'; l.href = a.href;
       document.head.appendChild(l);
     }, { once: true });
   });
   ```

3. **Host on a CDN-backed free tier** — Cloudflare Pages, Netlify, or Vercel. All three put the
   files on a global edge network for free. Avoid any host that serves from a single region.

---

## Priority 6 — Correctness and resilience (small, worth doing)

1. **Reveal animations depend on JS.** Every `.reveal` element starts at `opacity: 0`. If JS fails
   to run — old browser, blocked script, an aggressive extension — the page is blank below the hero.
   Add a `<noscript>` guard:
   ```html
   <noscript><style>.reveal,.reveal-left,.reveal-right{opacity:1!important;transform:none!important}</style></noscript>
   ```

2. **`content-visibility: auto` with `contain-intrinsic-size: auto 800px`** on sections causes
   scrollbar jumping as sections render. Measure the real heights and set per-section values instead
   of one blanket 800px.

3. **Add `fetchpriority="high"`** to the hero portrait (already done) and confirm nothing else
   competes with it — no other image should load eagerly.

4. **Add a `sitemap.xml` and `robots.txt`.** Three URLs, ~10 lines, helps the gallery subpages get
   indexed as their own results.

---

## Suggested execution order

| Step | Effort | Impact |
|---|---|---|
| 1. Enable Brotli/gzip at host + cache headers | 10 min | High |
| 2. Self-host and subset fonts | 45 min | Highest |
| 3. Generate small image variants + `srcset` | 60 min | High |
| 4. `content-visibility` on gallery `.shot` | 10 min | Medium |
| 5. Throttle nav scroll handler, cancel idle rAF | 20 min | Medium |
| 6. Hover-prefetch, `<noscript>` guard, sitemap | 20 min | Low |

Steps 1 and 2 together are most of the total gain.

---

## How to measure

Do not optimize blind. Before and after each step:

1. **Chrome DevTools → Lighthouse**, mobile preset, "Simulated throttling". Track LCP, CLS, TBT.
2. **Network tab, "Fast 4G" + "Disable cache"** — check total transferred bytes and the waterfall
   for anything blocking before first paint.
3. **PageSpeed Insights** (free, pagespeed.web.dev) once deployed — gives real-world Core Web Vitals
   from Chrome field data, which the local test cannot.

**Targets:** LCP < 2.0 s on mobile 4G, CLS < 0.05, total transfer for a first visit < 500 KB.

---

## Explicitly not recommended

- **A build step / bundler.** The site is 3 files. Adding Vite or a minifier costs maintenance and
  buys a few KB that compression already handles.
- **A JS image lightbox library.** Native `loading="lazy"` plus the gallery pages already cover it.
- **Converting to AVIF.** Encode times are long, Safari support has edge cases, and the WebP files
  are already small. Revisit only if image bytes remain the bottleneck after Priority 2.
- **Inlining images as data URIs.** Kills caching and inflates the HTML by ~33%.
