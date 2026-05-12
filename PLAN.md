# Upgrade & Refactor Plan

## Analysis

Small Astro SSG portfolio — one page, five components, ~12 public images. ~3 years stale.

### Current state
- `package.json`: Astro 3.5.5, `@astrojs/tailwind` 5, Tailwind 3.3, daisyUI 4.4, plus `autoprefixer` and `postcss-cli`
- `src/pages/index.astro`: inline `<head>`, hero text, `<Carousel>`, 4 social icons
- `src/components/{Github,LinkedIn,M19Y,Mail}.astro`: near-identical `<a><svg>…</svg></a>` files differing only in href / title / path-d
- `src/components/Carousel.astro`: daisyUI carousel, shuffles 7 images at build time
- `public/`: WebP set (`mru0..5`) plus three orphaned originals (`mru0.jpeg`, `mru4.jpeg`, `mru5.JPG`)
- `tsconfig.json`: aliases for `@layouts/*`, `@img/*`, `@styles/*` — none of those directories exist
- `tailwind.config.cjs`: only daisyUI is used as plugin; the daisyUI classes actually rendered are `carousel`, `carousel-item`, `rounded-box`
- `netlify.toml`: HSTS header only
- `README.md`: empty/placeholder

### Latest upstream
- astro 3.5.5 → **6.3.1** (3 majors behind)
- @astrojs/tailwind 5 → **6.0.2** (but deprecated path for TW4 — see below)
- tailwindcss 3.3.5 → **4.3.0** (CSS-first config, breaking)
- daisyui 4.4.1 → **5.5.19** (requires TW4)
- Node 24.11 locally — fine

### Issues found
1. **Dead deps:** `autoprefixer`, `postcss-cli` — no postcss config, no script uses them.
2. **Broken meta:** `og:image="https://www.maxrickettsuy/image.jpg"` — malformed host, image doesn't exist.
3. **Duplicated icon markup:** four files with identical wrapper / svg attrs / hover class.
4. **Stale tsconfig paths** for non-existent directories.
5. **Orphan public assets:** `mru0.jpeg`, `mru4.jpeg`, `mru5.JPG` unused; `mru.webp` used but inconsistent with `mru[0-5]` naming.
6. **No layout component** — head metadata lives in the page.
7. **Raw `<img>`** in Carousel — no optimization, no width/height (CLS), no `loading` / `alt`.
8. **Shuffle at build time only** — order is fixed once built, despite reading like client randomness. (Commit `Fix(?) shuffling` hints this was a sore spot.)
9. **Author / description fields** in `package.json` are empty / generic.

---

## Plan

### 1. Dependency upgrade
- Bump `astro` 3 → 6. No breaking features in use (no content collections, no image service config, no integrations beyond Tailwind).
- Replace `@astrojs/tailwind` with `@tailwindcss/vite` and adopt Tailwind 4's CSS-first config (`@import "tailwindcss"` in `src/styles/global.css`, `@plugin "daisyui"`). Delete `tailwind.config.cjs`.
- Bump `daisyui` 4 → 5. Only `carousel`, `carousel-item`, `rounded-box` used — all still in v5.
- Remove `autoprefixer` and `postcss-cli`.
- Clean up `package.json` metadata (drop `main: "index.js"`, add `"type": "module"`, drop redundant `start` alias).

### 2. Astro config + entry
- `astro.config.mjs`: swap the Tailwind integration for `vite: { plugins: [tailwindcss()] }`.
- Add `src/styles/global.css` with `@import "tailwindcss"; @plugin "daisyui";` and import it from a new `Layout.astro`.

### 3. Component refactor
- Add `src/layouts/Layout.astro` containing `<html>`, `<head>` (title, meta, canonical, favicon, theme-color, umami script), `<body class="bg-gray-900 text-white">`, `<slot/>`.
- Collapse the 4 icon components into one `SocialIcon.astro` taking `{ href, title, d, external? }`. Drive icons from a `socials.ts` data array in `index.astro`.
- Carousel: use Astro's `<Image>` (or at minimum add `width`, `height`, `alt`, `loading="lazy"` to each `<img>`). If true per-visit shuffle is desired, do it in a small inline `<script>`; otherwise keep build-time shuffle and note it.
- Fix og:image to a real asset (e.g. `https://www.maxrickettsuy.com/mru.webp`) and add og:image dimensions.

### 4. Cleanup
- Delete orphan public files: `mru0.jpeg`, `mru4.jpeg`, `mru5.JPG`. (Confirm before deleting.)
- Trim `tsconfig.json` paths to only resolving aliases (`@components/*`, `@layouts/*`, `@styles/*`); create the dirs that back them.
- Add a real README (one-liner: "Personal site, built with Astro + Tailwind + daisyUI. `npm run dev`.").

### 5. Verification
- `npm run build` clean, `npm run dev` opens the page, all four social links and the carousel render, hover color works, umami script still injected, OG meta validates.

### Out of scope without confirmation
- Deleting the orphan public images.
- Changing the shuffle semantics from build-time to client-time (behavioral change).
- Removing the `netlify.toml` HSTS header.
