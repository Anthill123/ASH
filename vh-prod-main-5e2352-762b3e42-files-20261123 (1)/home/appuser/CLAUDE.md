# Project: ASH Financial — static marketing site + Ash AI chat PWA

## Overview
ASH Financial (`www.ash-financial.com`) is a single-page financial-services marketing site: one hand-written `index.html` (223,621 B / 3,287 lines) plus a separate `/chat/` PWA that embeds a Microsoft Copilot Studio agent ("Ash AI Assistant"). **There is no build step, no Node process, no Docker, no backend** — nginx serves `/home/appuser/app` as a plain directory on port 80, and all dynamic behaviour (lead forms, intake wizard, animations) is inline vanilla JS talking to third-party endpoints (FormSubmit.co + a Google Apps Script web app). Editing the site = editing `index.html` in place; nginx needs no reload, only a browser refresh (HTML is served `no-cache, must-revalidate`).

Live preview: `https://vh-prod-main-5e2352-762b3e42.livemy.site` (200, 0 redirects, 223,720 B). Canonical production host declared in the markup: `https://www.ash-financial.com/`.

## Stack
| Layer | What is actually used |
|---|---|
| Markup | Hand-authored HTML5, single file `index.html` + `chat/index.html` |
| CSS | Inline `<style>` blocks (2: lines **97–1125** and **2836–2852**) + **Tailwind via CDN** `https://cdn.tailwindcss.com` (runtime JIT, **no PostCSS/tailwind.config.js, nothing compiled**) |
| Icons | `lucide` via `https://unpkg.com/lucide@latest` (runtime; icon names resolved from the CDN, e.g. `data-lucide="gauge"`) |
| Fonts | Google Fonts — Inter (300–700), Montserrat (400–900) |
| JS | Vanilla ES5-style inline scripts (**11** `<script>` elements on disk). No bundler, no framework |
| PWA | `/chat/sw.js` (cache `ash-ai-shell-v3`), `/chat/manifest.json`, root `site.webmanifest` |
| Server | nginx 1.18.0 (Ubuntu), static root, no `proxy_pass` anywhere |
| Host runtime | Ubuntu 24.04 LXC; node v22.22.2 + npm 10.9.7 and python3/docker/git installed but **unused by this project** |
| Package manager | none — no `package.json`, no `node_modules`, no lockfile |

Brand tokens: `--brand: #C8102E` (ASH red), `--brand-dark: #a00d24`, dark-section highlight gold `#e0b951`, ink `#0a0a0a`. Motion accents: `#ff8a9b`, `#ff6b80`, `#ff5c74`, `#fff0b8`.

## File Structure
```
/home/appuser/app/                 <- nginx root, owned appuser:appuser
├── index.html            223,621 B  THE site — all markup, CSS, JS in one file (3,287 lines)
├── CLAUDE.md              this file (untracked, not linked from any page)
├── chat/                            Ash AI Assistant PWA (own scope, own icons)
│   ├── index.html          9,352 B  shell + full-bleed iframe to Copilot Studio
│   ├── sw.js               1,527 B  service worker, scope ./ only (never touches the main site)
│   ├── manifest.json         417 B  standalone PWA manifest
│   ├── logo.png, favicon.svg, favicon.ico, icon-192.png, icon-512.png, apple-touch-icon.png
├── resources/                       ** LIVE PDFs — these are the ones linked from index.html **
│   ├── bookkeeping-checklist.pdf           md5 aa2d06dd…
│   ├── new-business-startup-checklist.pdf  md5 2d7446c5…
│   ├── tax-planning-checklist.pdf          md5 34d35269…
│   └── tax-preparation-checklist.pdf       md5 15132c41…
├── checklists/                      ** STALE duplicate set (Oct 3), NOT referenced anywhere **
├── assets/logo-ashai.png (246 KB), logo-ashai-light.png (69 KB)
├── favicon.ico / favicon.svg / favicon-96.png / icon-192.png / icon-512.png / apple-touch-icon.png
├── site.webmanifest, robots.txt, sitemap.xml
├── .gitignore
└── .git/                            local-only repo, no remote, single commit "faa8bf5 Initial deploy"
```
Backups live **outside** the web root in `/home/appuser/` as `index.html.bak-<tag>-<YYYYMMDD>` (verified 2026-11-19: **48** `index.html.bak-*` files, Sep 2026 → Nov 2026), plus `app.conf.bak-seo-20261015`, `icons.bak-20261004/`, `resources.bak-20261003/` — 51 backup entries in total. Never put a `.bak` or a helper `.py` inside `/home/appuser/app/` — it would be publicly served. Verified 2026-11-19: `find /home/appuser/app -name '*.py' -o -name '*.bak*' -o -name '*.sh'` → empty.

## Entry Points
| URL | Served from | Notes |
|---|---|---|
| `/` | `/home/appuser/app/index.html` | 200, 223,720 B over the wire (99 B larger than on disk — nginx `sub_filter` appends the preview-reload script) |
| `/chat/` | `/home/appuser/app/chat/index.html` | 200, 9,451 B; iframes Copilot Studio |
| `/resources/*.pdf` | static files | delivered after a lead form is submitted |
| `/sitemap.xml`, `/robots.txt`, `/site.webmanifest` | static | all point at `www.ash-financial.com`, not the preview host |
| anything else | `try_files $uri $uri/ $uri.html /index.html` | SPA-style fallback to the homepage |

## Scripts
**There are no npm scripts — no `package.json` exists.** Nothing is built, bundled, minified or transpiled. The deploy "build" is: write `index.html` → nginx serves it on the next request.

Useful real commands instead:
```bash
curl -s -o /dev/null -w '%{http_code} %{size_download}\n' http://localhost/        # 200 223720
curl -s -o /dev/null -w '%{http_code} %{size_download}\n' http://localhost/chat/   # 200 9451
sudo nginx -t && sudo systemctl reload nginx                      # only after editing app.conf
cp /home/appuser/app/index.html /home/appuser/index.html.bak-<tag>-$(date +%Y%m%d)   # ALWAYS before an edit
```

## Key Dependencies
All dependencies are **runtime CDN / third-party services** — there is nothing installed on the VM.

| Dependency | Purpose | Probe result (2026-11-17) |
|---|---|---|
| `cdn.tailwindcss.com` | Tailwind utility classes, compiled in the browser | 302 (alive) |
| `unpkg.com/lucide@latest` | icon set, `lucide.createIcons()` at load | 302 (alive) |
| `fonts.googleapis.com` / `fonts.gstatic.com` | Inter + Montserrat | alive |
| `formsubmit.co/ajax/contact@ash-financial.com` | **source of truth** for every lead → email | 200 at root at a 25 s timeout (405 on the ajax path for GET — POST-only, normal) |
| `script.google.com/a/macros/ash-financial.com/s/AKfycbz…/exec` | Google Sheets mirror of leads (fire-and-forget, `mode:'no-cors'`) | 302 (alive) |
| `copilotstudio.microsoft.com` | the Ash AI chat agent iframe | 200 (alive) |
| `www.amazon.com/dp/...` | book purchase links | outbound links only |

## Routes / Pages
Single page, anchor-navigated. 14 `<section>` elements in `index.html` (id → purpose):

| `#id` | `data-id` | Heading / purpose |
|---|---|---|
| `#home` | 37 | Hero — "Building Wealth. Creating Legacy." (`#hero-content-m3n4` = the `.hero-content` div; CTA row is `[data-id="44"]`) |
| `#trust-strip` | ts0 | Trust badges strip (`.trust-num`, `.trust-stars`); `ts2n` counter is `data-count="6"` → "Core Service Areas" |
| `#value-prop` | vp0 | Value proposition (`vp3` is scroll-revealed) |
| `#about` | ab0 | Our Story |
| `#mission` | 47 | Mission |
| `#philosophy` | ph0 | Philosophy |
| `#services` | 67 | **Six** Core Service Categories — grid `#services-grid-y5z6` (`data-id="72"`), cards `sc1`…`sc6` |
| `#why-us` | wc0 | Why Choose ASH Financial (`wc5` = checklist rows, hover + reveal) |
| `#how-we-serve` | bm0 | How ASH Financial Serves Clients |
| `#who-we-help` | wh0 | Who We Help (`wh6` = checklist rows, hover + reveal) |
| `#testimonials` | tm0 | What Clients Say — REVIEWS 2.0 cards (`tm6` container; reviewer lines `tm7n`/`tm8n`/`tm9n` carry CSS-generated initials **GG / BW / TR**; incl. "Terry Robinson") |
| `#books` | 108 | The Money Blueprint Series (4 Amazon links: `B0HJQ62RYF`, `B0HLW1RWTG`, `B0HNM56KZH`, `B0GYK6L3P7`) |
| `#resources` | cr0 | Free Resources & Checklists (4 gated PDFs) |
| `#contact` | 168 | Contact form + `#intake` multi-step client-intake wizard |

**Service cards (`#services-grid-y5z6`, in DOM order):** `sc1`…`sc5` (the original five categories) + `sc6` **Credit Analysis & Improvement** (lucide icon `gauge`, five bullets: Credit Report Review, Credit Score Analysis, Dispute & Error Guidance, Debt & Utilization Strategy, Credit Building Roadmap, plus a disclaimer `sc6note`: *"Educational credit guidance only. ASH Financial is not a credit repair organization and does not file disputes on a client's behalf."*). `sc6` is the last child of the grid.

Service checkbox chips (kept in sync with the cards): contact form `.cf-service` — **7** chips (credit = `data-id="205c"`); intake wizard `.ik-service` — **9** chips (credit = `data-id="ik22c"`, `data-key="credit"`). Both credit chips sit immediately **before** their "Other" sibling. The credit intake chip deliberately has **no** `ik-detail-credit` panel, so `detailKeys()` skips it and it flows straight through `services()` into the review screen and the lead email.

Sticky mobile section nav: `<nav id="mobile-tabs-mt0" class="mtabs">` with `.mtab[data-scroll]` → Home / About / Mission / Services / Reviews / Books (+ more), kept in sync by `centerTab()` / `onScroll()`.

JS-injected chrome (no markup in the file): `#scroll-progress` (top reading bar), `#to-top` (back-to-top, **left** side — `#ashChatBtn` owns bottom-right), `.scroll-cue` (hero scroll hint, desktop only), `.hero-grid` + `.hero-stage` with `.hero-orb` children (hero 3D scene), and one `.dk-layer` with `.dk-orb` children injected into each of 7 desktop sections.

Second page: `/chat/` — header + install button + `<iframe id="chatFrame">` pointing at Copilot Studio environment `e7efbfde-51ed-e3f5-93d9-56ca968a800f`, bot `cr8f7_ashai_pWNnCq`.

## Configuration

### nginx — `/etc/nginx/sites-available/app.conf` (2,343 B, symlinked into `sites-enabled/`, this is the FULL active config; mtime 2026-10-15 17:41, untouched since)
```nginx
# livemy.app — ASH Financial (plain static HTML site, no backend)
# Container nginx serves plain HTTP on :80 only.
# TLS is terminated by the bastion HAProxy: Browser -> Cloudflare -> HAProxy:443 -> here:80.
# NO `listen 443 ssl`, NO ssl_certificate under /tmp/.

map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

# External traffic from the bastion is https; localhost healthchecks are plain http.
# Any future proxy_pass block must use $vh_fwd_proto — never $scheme (causes a 301
# loop through Cloudflare). This site is static, so nothing is proxied today.
map $remote_addr $vh_fwd_proto {
    127.0.0.1   http;
    ::1         http;
    default     https;
}

# SEO: the livemy.site preview hostname must never be indexed — the public site
# is https://www.ash-financial.com/ and index.html already declares it canonical.
# Empty value => nginx omits the header entirely on the real domain.
map $host $vh_robots {
    default                 "";
    "~*\.livemy\.site$"     "noindex, nofollow";
}

server {
    listen 80;
    server_name vh-prod-main-5e2352-762b3e42.livemy.site _;

    root /home/appuser/app;
    index index.html;

    # Inject a postMessage listener into every HTML response so the parent preview
    # iframe can reload itself after an AI edit.
    # sub_filter_types already covers text/html by default — listing it again warns.
    sub_filter '</body>' '<script>window.addEventListener("message",e=>{if(e.data==="vh-reload")location.reload()});</script></body>';
    sub_filter_once off;

    # The git metadata and any dotfiles live next to index.html — never serve them.
    location ~ /\. {
        deny all;
        return 404;
    }

    # HTML must always be revalidated so AI-assisted edits show up without a hard refresh.
    location / {
        try_files $uri $uri/ $uri.html /index.html;
        add_header Cache-Control "no-cache, must-revalidate" always;
        add_header X-Robots-Tag $vh_robots always;
    }

    location ~* \.html$ {
        add_header Cache-Control "no-cache, must-revalidate" always;
        add_header X-Robots-Tag $vh_robots always;
    }

    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff2?)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header X-Robots-Tag $vh_robots always;
    }
}
```
Verified 2026-11-19: `nginx -T | grep -E '^\s*(listen 443|ssl_certificate)'` → **empty** (no TLS inside the container, no `/tmp/` cert paths). No `proxy_pass` exists, so `$vh_fwd_proto` is currently unused but is pre-declared for any future backend. Config backup `/home/appuser/app.conf.bak-seo-20261015` is the **pre-SEO** version — it lacks the `$vh_robots` map and the three `X-Robots-Tag` lines; that diff is expected, not drift. `/etc/nginx/sites-available/default` still exists on disk but is **not** symlinked into `sites-enabled/` (`sites-enabled/` contains only `app.conf` → `/etc/nginx/sites-available/app.conf`).

### Environment variables
**None.** No `.env`, `.env.production` or `.env.local` anywhere in the project. Every secret-looking value (Apps Script URL, Copilot bot id) is intentionally public and baked into the HTML.

### Build output
None. The served directory *is* the source directory (`/home/appuser/app`).

### systemd units
Only platform/base units — this project adds none. `nginx.service`: **enabled + active**. Also running: `vibhost-agent` (vh-agent, port 50051), `docker.service`, `containerd.service`, `fail2ban.service`, `ssh.service`, `cron.service`, `snapd.service`, `rsyslog.service`, `systemd-resolved`, `unattended-upgrades.service`, `user@1001.service`, `polkit`, `packagekit`, `dbus`, `networkd-dispatcher`, `console-getty`.

## Runtime State (verified 2026-11-19 20:48 UTC)
- **Uptime:** 18 days, 8:05. Load 0.31 / 0.44 / 0.44.
- **Listening ports:** `:80` nginx (pids 442/451), `:22` sshd (pid 357), `*:50051` vh-agent (pid 328), `127.0.0.1:45537` containerd (pid 330), `127.0.0.53:53` systemd-resolved. **Nothing else — no app process.**
- **Docker:** daemon running, **zero containers** (`docker ps` empty).
- **No node / python / pm2 / gunicorn process belongs to this project.**
- **Disk:** `/` and `/home` on `/dev/loop8` — 1.2 T total, 75 G used, 1.1 T free (7 %).
- **Memory:** 32,095 MB total, 9,862 MB used, 21,637 MB available, no swap.
- **Health:** `curl localhost/` → 200 / 223,720 B; `curl localhost/chat/` → 200 / 9,451 B; bastion probe (`Host: vh-prod-main-5e2352-762b3e42.livemy.site` + `X-Forwarded-Proto: https`) → `HTTP/1.1 200 OK`, `Server: nginx/1.18.0 (Ubuntu)`, `Cache-Control: no-cache, must-revalidate`, `X-Robots-Tag: noindex, nofollow`. Public URL → 200, 0 redirects, 223,720 B; public `/chat/` → 200, 0 redirects.
- **index.html fingerprint:** 223,621 B, 3,287 lines, md5 `0acf9112e57c1b4e34da50a6a88f058c`, inode **46335** (preserved across every edit), `appuser:appuser`, mode 644, mtime 2026-11-19 20:46.
- **Tag census on disk:** `<style>` 2 · `<script` 11 · `<section` 14 · `<div` 260 · `data-id=` 789 (markup anchors + CSS attribute selectors + JS strings — see note 19).
- **git:** repo at `/home/appuser/app/.git`, no remote, one commit `faa8bf5 Initial deploy`; `index.html` shows as modified and everything added after that commit (`CLAUDE.md`, `chat/`, `assets/`, `resources/`, icons, sitemap…) is untracked. Nothing is committed on edit — **git is not the backup mechanism here; the `.bak` files are.** Running git as root hits `detected dubious ownership` — use `su appuser -c 'cd /home/appuser/app && git …'`.

## Credentials & Services
No passwords, API keys or secrets exist on this VM for this project — the site is 100 % static and every integration is anonymous/public by design. Nothing to rotate.

| Service | Identifier / endpoint | Auth |
|---|---|---|
| Lead email (source of truth) | `https://formsubmit.co/ajax/contact@ash-financial.com` | none — public POST endpoint, keyed by the destination address |
| Google Sheets mirror | `https://script.google.com/a/macros/ash-financial.com/s/AKfycbzID65wztabPEUNEd5Gyi6yiggFPHljFMbzO1FE_ia9a3f0aDnvTHUX6ESdnneKCxso/exec` | none — anonymous Apps Script web app, `mode:'no-cors'`, fire-and-forget |
| Ash AI chat | Copilot Studio env `e7efbfde-51ed-e3f5-93d9-56ca968a800f`, bot `cr8f7_ashai_pWNnCq`, `?__version__=2&enableFileAttachment=false&cliAgent=true&v=20261013` | anonymous web chat |
| Business contact | `contact@ash-financial.com` | — |
| Sheet tabs written by the site | `Contact`, `Checklists`, `Intake` | — |
| Email subjects | `New Consultation Request from …`, `Checklist Request — …`, `New Client Intake — …` | — |
| Public preview host | `https://vh-prod-main-5e2352-762b3e42.livemy.site` | — |
| Production host (canonical) | `https://www.ash-financial.com/` | — |
| OS account | `appuser` (uid 1001, sudo). No password set or changed by any deploy; SSH key auth only | — |
| Browser storage keys | `localStorage['ashIntakeV1']` (intake wizard draft), `localStorage['ashAiHintSeen']` (iOS install hint) | — |

## Deploy History

### 2026-11-19 (20:46 UTC) — SERVICES: added "Credit Analysis & Improvement" (sixth category)
All five edits are live and verified.

**Status:** ✅ Working · https://vh-prod-main-5e2352-762b3e42.livemy.site

**Changed:** `/home/appuser/app/index.html` only — added the "Credit Analysis & Improvement" service: the `sc6` card as last child of `services-grid-y5z6`, heading `Five`→`Six Core Service Categories`, trust counter `data-count` 5→6, plus one `cf-service` chip (`205c`) and one `ik-service` chip (`ik22c`), both placed before their "Other" sibling. 220,331 → 223,621 B. No build (this project has none — nginx serves the directory, live on write), no nginx reload, no other file touched.

**Verified — real output:**

| Check | Result |
|---|---|
| `curl` `/` · `/chat/` | **200** · **200** |
| `curl -s localhost/ \| wc -c` | **223720** (> 222500) |
| served: `sc6` · `gauge` · `Six Core…` · `data-count="6"` · `data-key="credit"` | **1 · 1 · 1 · 1 · 1** |
| served: `Five Core Service Categories` · `sc1`…`sc5` | **0** · **1 each** |
| unchanged markers | `HERO 3D DESKTOP` 1 · `REVIEWS 2.0` 1 · `DESKTOP MOTION` 2 · `HERO 3D LAYER` 2 · `DYNAMIC LAYER` 2 · `FLASH LAYER` 1 · `class="hl` 3 · `trust-strip` 1 · `formsubmit.co/ajax/` 3 · `sendToSheet(` 4 · `Terry Robinson` 1 · `mobile-tabs-mt0` 2 · `ashChatBtn` 1 · `MSA` 9 — **all exact** |
| chips in markup | `cf-service` 6→**7**, `ik-service` 8→**9** |
| disk tags | `<style>` 2/2 · `<section>` 14/14 · `<script>` 11/11 · `<ul>` **+1** |
| parser / CSS | `html.parser` **0 errors**; style block 1 braces **375/375**, identical to backup |
| `diff` vs backup | `1225c` `1350c` `1423a1424,1438` `1898a1914` `2128a2145` — **0 lines deleted** |
| identity | inode **46335** preserved, `appuser:appuser`, `644`, last byte `0x3e` |
| public URL | **200, 0 redirects**, `Credit Analysis` and `Six Core Service Categories` present; bastion probe `HTTP/1.1 200 OK` |

The helper ran a purity proof before writing: stripping the inserts and reverting the two replacements reproduced the original bytes exactly, so no surrounding line drifted. `sc6` is confirmed the last of six cards in the grid, closing straight into the grid's `</div>`. `gauge` is a valid lucide icon name; the attribute is present, and since lucide resolves it from the CDN at runtime, visual rendering isn't checkable from a shell.

**Three predicted counts came out differently — arithmetic, not edit errors:**

1. **`Credit Analysis &amp; Improvement` → 5, not 3.** The prediction counted the card heading (1) + contact chip (2 — value attribute and visible span). The intake chip from Edit E carries the same phrase twice for the same reason, so 1 + 2 + 2 = 5.
2. **`<div>`/`</div>` → +3, not +2.** The card block contains three divs: the `sc6` card wrapper, the `sc6a` icon wrapper and the `sc6d` divider. Balanced at 260/260.
3. **`cf-service` raw grep 7→8, `ik-service` 10→11.** The baselines were already above the stated 6 and 8 because the JS selector strings `.cf-service:checked` (×1) and `.ik-service:checked` (×2) also match. Counting actual chip markup gives exactly the predicted 7 and 9.

Also worth noting: the brief's summary line said "four pure inserts", but its own A–E list specifies three inserts and two single-line replacements — the diff shows exactly that (3 `a` hunks + 2 `c` hunks). No markup was adjusted to force any number.

**Backup retained:** `/home/appuser/index.html.bak-credit-20261119` — 220,331 B, md5 `02a774ef6c2e3248e3b0b566028a76f5`, verified identical to the pre-edit file. Restore with `cp` to keep the live inode. Helper deleted — `find /home/appuser/app -name '*.py' -o -name '*.bak*' -o -name '*.sh'` returns empty.

**Credentials:** none introduced — no form, Sheets, chat or CSS code was touched, and the new intake chip deliberately has no `ik-detail-credit` panel, so `detailKeys()` skips it and it flows straight through `services()` into the review screen and email.

### 2026-11-17 (20:52 UTC) — DESKTOP MOTION: ambient section depth + scroll parallax + card tilt
All checks pass. Every predicted count landed exactly as specified — nothing needed explaining away.

**Status:** ✅ Working · https://vh-prod-main-5e2352-762b3e42.livemy.site

**Changed:** `/home/appuser/app/index.html` only — two pure inserts: a 40-line `DESKTOP MOTION` CSS block at the end of the first `<style>` (lines 1085–1124, after `REVIEWS 2.0`) and a 101-line `DESKTOP MOTION JS` `<script>` before `</body>` (lines 3167–3267). 212,938 → 220,331 B. No build (this project has none — nginx serves the directory, so the edit was live on write), no nginx reload, no other file touched.

What it adds. CSS (all inside `@media (min-width: 769px)` except the keyframes and the reduced-motion block): `#value-prop, #services, #why-us, #how-we-serve, #who-we-help, #books, #resources { position: relative; isolation: isolate }` so each section owns its stacking context; `.dk-layer` (absolute `inset: 0`, `z-index: -1`, `overflow: hidden` **on the layer, never the section**, `translate3d(0, var(--dk-y), 0)`); `.dk-orb` (blurred radial-gradient circles, `dkFloat var(--t) ease-in-out infinite alternate`); faster `transform` transitions on `.service-card` / `.book-cover` so the JS tilt reads smoothly; `@keyframes dkFloat`; and a `prefers-reduced-motion` block killing orb animation and layer transform. JS (IIFE, returns immediately under 769 px or with reduced motion): builds a `.dk-layer` + 1–2 `.dk-orb`s into each of the 7 sections from a `SECTIONS` table (per-section depth `d` 0.9–1.3), a rAF-throttled scroll/resize `paintDk()` writing `--dk-y` up to `-70 × d` px, an idle hero drift loop feeding `--hero-mx` / `--hero-my` (started/paused by an `IntersectionObserver` on `.hero-section`, reset by `pointermove`), and a `perspective(900px) rotateX/rotateY` pointer tilt on `#services .service-card, #how-we-serve .service-card, #resources .service-card, .book-cover` (skipped on `(hover: none)`; `pointerleave` clears the inline transform).

**Verified — real output:**

| Check | Result |
|---|---|
| `curl` `/` · `/chat/` | **200** · **200** |
| `curl -s localhost/ \| wc -c` | **220430** (> 219000) |
| served: `DESKTOP MOTION` · `dkFloat` | **2** · **2** |
| served: `dk-layer` · `dk-orb` · `--dk-y` | **3** · **3** · **2** (≥3/≥3/≥2) |
| served: `isolation: isolate` · `perspective(900px)` | **1** · **1** |
| unchanged markers | `HERO 3D DESKTOP` 1 · `REVIEWS 2.0` 1 · `HERO 3D LAYER` 2 · `DYNAMIC LAYER` 2 · `FLASH LAYER` 1 · `class="hl` 3 · `trust-strip` 1 · `formsubmit.co/ajax/` 3 · `sendToSheet(` 4 · `Terry Robinson` 1 · `mobile-tabs-mt0` 2 · `ashChatBtn` 1 · `MSA` 9 — **all exact** |
| disk tags | `<style>` 2/2 · `<section>` 14/14 · `<script>` **11/11** (was 10/10) · `<div>` 257/257 · `data-id=` 768 — unchanged vs backup |
| braces | style#1 375/375, style#2 18/18, end-depth 0, never negative; `@media` 25→27 (+2 new, balanced); new JS `{}` 39/39, `()` 119/119, `[]` 34/34; `html.parser` **0 errors** |
| `diff` vs backup | **`1084a1085,1124` + `3126a3167,3267`** — 2 append hunks, **0 removed, 0 changed** |
| file identity | inode **46335** preserved, `appuser:appuser`, `644`, last byte `0x3e`, no trailing newline |
| hygiene | helper deleted; `find … '*.py' -o '*.bak*' -o '*.sh'` → **empty**; only `index.html` has a new mtime; `chat/` and nginx config untouched |
| public URL | **200, 0 redirects**, 220,430 B, `DESKTOP MOTION` ×2; `/chat/` 200, 0 redirects; bastion probe `HTTP/1.1 200 OK` |

Beyond the required list, the helper asserted the inserts were *provably* pure: stripping the two blocks from the result reproduced the original bytes exactly, so no existing line could have drifted. It also pre-checked the counts before writing and refused on any mismatch.

**Backup retained:** `/home/appuser/index.html.bak-dkmotion-20261117` — 212,938 B, md5 `f1c3870e2a0d88cb134f5bb03defdf9e`, verified identical to the pre-edit file. Restore with `cp` to keep the live inode.

**Credentials:** none — this change introduces no keys, tokens or secrets, and no form, Sheets or chat code was touched.

### 2026-11-17 (18:11 UTC) — REVIEWS 2.0 *(reconstructed from the VM — no report was filed)*
Backup `/home/appuser/index.html.bak-rev3d-20261117` (206,976 B, `appuser:appuser`, 18:11) is the state **before** this change; it produced the 212,938 B file. Added the `REVIEWS 2.0: cards with depth, quote marks and initials` CSS block, now **lines 1000–1084** of style block 1 (85 lines, the last layer before DESKTOP MOTION). It restyles `#testimonials` only, entirely through `data-id` attribute selectors so no markup moved: a white→grey vertical gradient on the section (`!important`), `tm6` grid gap 28 px with stretched cards, depth/shadow + lift-on-hover on `tm6 > div`, a brand bar `::before` replacing the flat 3 px red top border, an oversized watermark quotation mark `::after`, larger gold-glow stars, a larger/darker quote body, brand-red fading dividers (`tm7d`/`tm8d`/`tm9d`), and **CSS-generated initials avatars** on the reviewer lines — `tm7n::before { content: "GG" }`, `tm8n::before { "BW" }`, `tm9n::before { "TR" }` (the reviewer `<br>` is hidden and `tm7g`/`tm8g`/`tm9g` become block elements so the line still wraps). Includes a `@media (max-width: 600px)` tightening pass and a `prefers-reduced-motion` block that also neutralises the hero 3D transforms. No build, no nginx reload, no other file touched.

### 2026-11-17 (18:09 UTC) — HERO 3D LAYER + HERO 3D DESKTOP *(reconstructed from the VM — no report was filed)*
Backup `/home/appuser/index.html.bak-hero3d-20261117` (199,934 B, 18:09) is the state **before** this change — byte-identical in size to the post-DYNAMIC-LAYER file, confirming this edit sat directly on top of the 2026-11-16 19:05 deploy; it produced the 206,976 B file. Two CSS layers plus one script:
- **`HERO 3D LAYER` (lines 895–964)** — `.hero-section { perspective: 1100px; perspective-origin: 50% 42%; overflow: hidden }` turns the hero into a 3D scene; `#hero-content-m3n4` tilts with the pointer and drifts on scroll; `.hero-stage` + `.hero-orb` (`@keyframes orbDrift`) form a parallaxed depth field; `.hero-grid` is a perspective grid floor (`@keyframes gridRun`, shorter/dimmer under 768 px); `.hero-watermark` gets `wmSpin3D` (re-declaring the DYNAMIC LAYER's `wmFloat`); a `prefers-reduced-motion` block disables orbs, grid, watermark, stage and hero-copy transforms.
- **`HERO 3D DESKTOP` (lines 965–999, `@media (min-width: 769px)`)** — closer camera (`perspective: 850px`), deeper tilt + Z push on the copy, `h1` on `translateZ(48px)` and the CTA row `[data-id="44"]` on `translateZ(26px)`, brighter/harder-parallaxing orbs, a taller faster grid floor, 14 s watermark spin.
- **`HERO 3D LAYER` JS (block starting line 3089)** — injects `.hero-grid` and `.hero-stage` with its orbs into `.hero-section`, runs a rAF pointer-tilt loop (`pointermove` / `pointerleave`, skipped on `(hover: none)`), and a passive scroll handler feeding the same CSS vars so tilt and scroll drift compose.

### 2026-11-16 (19:05 UTC) — DYNAMIC LAYER: progress bar, parallax, reveals, ripples
Edited `/home/appuser/app/index.html` only — **two pure inserts**: a `DYNAMIC LAYER` CSS block (98 lines, now **797–894**, at the end of style block 1 right after `FLASH LAYER`) and a new `<script>` block (99 lines, then 2757–2855, before `</body>` at 2859). 189,635 → 199,934 B; 2,662 → 2,859 lines. Backup `/home/appuser/index.html.bak-dyn-20261116` (189,635 B, md5 `3e5d69f64cf9fac7be498501faa35947` — md5-identical to the pre-edit file). Applied with a one-shot `python3 -I` helper at `/home/appuser/_dyn.py` (dry run, then `--apply`), **deleted afterwards — confirmed gone**. No build, no nginx reload, no other file touched.

What it adds. CSS: `@keyframes heroWeave` (32 s background drift on `.hero-section`), `wmFloat` (`.hero-watermark`), `cueDot`, `rippleOut`, `chatPulse` (`#ashChatBtn`, 3.4 s, 2.4 s delay); a red drop-shadow glow on `#hero-content-m3n4 h1`; `#scroll-progress` reading bar; `.scroll-cue` (+`::after`, `:hover`, `.hidden`, hidden under 768 px); `#to-top` (left side, `.show` at y > 600); the generic `.rv`/`.rv.shown` reveal; hover states for `#why-us [data-id="wc5"] > div`, `#who-we-help [data-id="wh6"] > div` (icon `scale(1.18)`), `#testimonials [data-id="tm6"] > div` (lift + brand glow); `.nav-wrap .nav-link::after` underline wipe; `.ripple`; and a `prefers-reduced-motion` block killing all of it. JS (IIFE): rAF-throttled `paint()` driving the bar / back-to-top / cue / hero parallax, an `IntersectionObserver` reveal over 9 selectors with a 4 s fail-safe that force-shows everything, and a delegated click ripple that self-removes after 650 ms.

Verified (real output): `/` 200, `/chat/` 200; `curl localhost/ | wc -c` = 200,033; public URL and `/chat/` both 200 with 0 redirects; bastion probe `HTTP/1.1 200 OK` — no redirect loop. Markers `heroWeave` 2 · `wmFloat` 2 · `cueDot` 2 · `rippleOut` 2 · `chatPulse` 2. Unchanged: `FLASH LAYER` 1 · `class="hl` 3 · `id="trust-strip"` 1 · `formsubmit.co/ajax/` 3 · `sendToSheet(` 4 · `Terry Robinson` 1 · `mobile-tabs-mt0` 2 · `id="ashChatBtn"` 1. Tags on disk: `<style>` 2/2 · `<section` 14/14 · `<div>` 257/257 · `<script>` **9/9** (was 8/8). Braces: style#1 310/310, style#2 18/18, end depth 0 never negative; `@media` 18→20 balanced; new script `{}` 34/34, `()` 96/96, `[]` 5/5; `html.parser` 0 errors. `diff` vs backup = exactly two hunks `796a797,894` + `2658a2757,2855` — 0 lines removed, 0 changed. inode 46335 preserved, `appuser:appuser`, 644, last byte `0x3e`, helper deleted, `find … -name '*.py' -o -name '*.bak*' -o -name '*.sh'` → empty. `app.conf` mtime still 2026-10-15 17:41; `chat/index.html`, `chat/sw.js`, `site.webmanifest`, `sitemap.xml` mtimes unchanged. No credentials introduced.

Count deltas that are selector/comment text, not markup: `DYNAMIC LAYER` 2 (one header comment per block) · `scroll-progress` 3 (rule + reduced-motion + `bar.id`) · `to-top` 5 (3 rules + `toTop.id` + the hyphenated word in the JS comment) · `scroll-cue` 7 (6 CSS + `cue.className`) · `data-id=` 719→736 = **+15 CSS selectors +3 JS strings; real markup anchors 718 before and 718 after** · `ashChatBtn` 7→10 (+3 from the new CSS rule, reduced-motion line and comment — the markup `id="ashChatBtn"` and all chat JS untouched).

**Correction to the previous report of this change.** It flagged the hero copy parallax/fade as non-rendering, on the theory that `.hero-content { animation: heroFadeIn 1.2s ease forwards }` (line 197) fills `opacity`/`transform` and outranks the inline style. That is not the live cascade: the ATTENTION & POLISH LAYER re-declares `.hero-content { animation: none; opacity: 1 }` at **line 634**, outside any `@media` (the mobile block closes at 629), same specificity and later in source — so it wins, `.hero-content` carries **no** filling animation, and `heroContent.style.transform` / `.opacity` apply normally (inline style also beats that rule's `opacity: 1`). The hero parallax works; entrance animation now belongs to the staggered children (`#hero-content-m3n4 > *` / `heroRise`). **No caveat stands for this deploy.**

### 2026-11-16 (18:35 UTC) — FLASH LAYER: page-wide shine/motion pass
Edited `/home/appuser/app/index.html` only — **one pure insert** of 81 lines / 4,369 B at the end of the first `<style>` block (then lines 716–796, directly before the untouched anchor). No build, no nginx reload, no other file touched. Backup `/home/appuser/index.html.bak-flash-20261116` (185,266 B, md5 `e64db7fa881362e9ba0798b4c7e385ee`). Applied with a one-shot `python3 -I` helper at `/home/appuser/_flash.py` (dry run, then `--apply`), deleted afterwards — confirmed gone.

What the block adds: `@keyframes flashSweep` / `heroSheen` / `glowBreathe`; a brand-tinted gradient sweep on `#hero-content-m3n4 h1`; a breathing `.hero-section::before` spotlight plus a new decorative `.hero-section::after` sheen (`z-index: 0`, `pointer-events: none`); gradient fill + brand glow and hover-lift on `.btn-red, .btn-primary`; a glowing animated hairline `.nav-wrap::after` revealed by `.nav-wrap.scrolled`; colored hover glows on `.service-card`, `.soon-card`, `.book-cover`; gradient `.trust-num` + glowing `.trust-stars`; **`.hl` / `.hl-dark` returned to a gradient (red / gold) with the `flashSweep` animation**; and a `prefers-reduced-motion` block that kills all of it.

Verified (real output): `/` 200, `/chat/` 200; `curl localhost/ | wc -c` = 189,734 (> 1000); `FLASH LAYER` ×1, `flashSweep` ×4, `heroSheen` ×2, `glowBreathe` ×2; `class="hl` ×3, `id="trust-strip"` ×1, `formsubmit.co/ajax/` ×3, `sendToSheet(` ×4, `Terry Robinson` ×1, `mobile-tabs-mt0` ×2; `<style>` 2/2, `<section` 14/14; brace balance of both style blocks 263=263 and 18=18 (end depth 0, never negative); `@media` 16 opens / all balanced (block 1 was 15, +1 new); `diff` vs backup = one hunk `715a716,796`, 0 removed, 81 added; inode 46335 preserved, `appuser:appuser`, mode 644; no helper or stray file in the web root; public URL 200 with 0 redirects and `FLASH LAYER` ×1. Cascade pre-check: every overridden selector is defined earlier in the same block, so the new rules win at equal specificity; `--brand` is in `:root`; `.hero-section` is `position: relative` with `.hero-content` at `z-index: 1`, above the new `z-index: 0` sheen; no pre-existing `.hero-section::after` or `.nav-wrap::after` was overwritten; `html.parser` reports 0 errors.
Two counts that look off are measurement artifacts, both reproduced on the untouched backup: `<script` 9 over the wire vs 8 on disk (nginx `sub_filter` injects the preview-reload script), and `<div` 254 by `grep -c` (lines) vs 257 by occurrence. nginx was not touched — `app.conf` mtime still 2026-10-15.
No credentials introduced.

### 2026-11-16 — Key-point highlight restyle (`.hl` / `.hl-dark`)
Edited `/home/appuser/app/index.html` only — 2 hunks: the `.hl`/`.hl-dark` rules (now lines 671–679) swapped from a marker-swipe gradient to bold brand-colored text (`font-weight:800; color:var(--brand); letter-spacing:.15px`, with `.hl-dark { color:#e0b951 }` for dark sections), and the now-dead `.hl` line removed from the `prefers-reduced-motion` block. No build, no nginx reload. Backup `/home/appuser/index.html.bak-hl-20261116` (md5 `8d922f88a53468802fbd24cb588e717c`, 185,571 B). Applied via a one-shot `python3 -I` helper (dry-run first, both anchors asserted exactly once, `r+b`+`truncate`); helper deleted afterwards.
Verified: `/` 200, `/chat/` 200; `class="hl` ×3; `background-size: 0% 38%` ×0; `Key-point highlight` ×1, `#e0b951` ×1; `formsubmit.co/ajax/` ×3, `sendToSheet(` ×4; `Terry Robinson` ×1, `mobile-tabs-mt0` ×2, `id="trust-strip"` ×1; `<style>` 2/2, `<script` 8/8, `<div>` 257/257 unchanged vs backup; `<style>` brace balance 235/235 and 18/18 (count fell 237→235 because each removed rule held one `{`/`}` pair — balance still exact); inode 46335 preserved, owner `appuser:appuser`, mode 644; helper deleted, no `.py`/backup left in the web root; public URL and `/chat/` both 200 with 0 redirects. Served size 185,365 B vs 185,266 B on disk = the nginx `sub_filter` preview-reload script, pre-existing.
No credentials introduced.

### Earlier (reconstructed from `/home/appuser/*.bak-*` — exact dates are the backup suffixes)
Append-only trail of every prior AI edit; each entry is the state *before* that change.
- 2026-11-16 `index.html.bak-polish-20261116` (176,777 B) — copy polish pass
- 2026-11-10 `bak-mobtabs-20261110` (172,498 B) — sticky mobile section tabs
- 2026-11-09 `bak-mobile-20261109` (169,755 B) — mobile layout work
- 2026-10-20 `bak-review3-20261020` (168,097 B), 2026-10-17/20 `bak-book4-20261020` (165,381 B) — testimonials + 4th book
- 2026-10-15/17 `bak-brandseo-20261017` (163,397 B), `bak-sheets-20261015` (162,126 B) — Google Sheets mirror + brand/SEO; `app.conf.bak-seo-20261015` is the matching nginx backup (adds the `X-Robots-Tag` map, sitemap/robots pointing at the production domain)
- 2026-10-14 `bak-canonical-20261014` (162,183 B), `bak-wording-20261014` (161,887 B) — canonical URL + copy
- 2026-10-12/13 `bak-booking-20261013` (155,656 B), `bak-newbot-20261012` (155,656 B), `bak-chat-20261012` (151,545 B) — Copilot Studio bot swap, `/chat/` PWA, booking
- 2026-10-04 `bak-logo-20261004` (150,530 B), `bak-book3-20261004` (147,686 B), `icons.bak-20261004/` — logos, icon set, 3rd book
- 2026-10-03 (busy day) `bak-datefix`, `bak-lead`, `bak-pdftab`, `bak-btnpdf`, `bak-resources`, `bak-pdfgate`, `bak-resforms`, `bak-emoji`, `bak-phone`, `bak-booksrework` (110 K→147 K) — the whole gated-PDF resources section + lead capture; `resources.bak-20261003/` is the PDF snapshot
- 2026-10-01/02 `bak-notemplates`, `bak-noqb`, `bak-services5`, `bak-legalname` — services trimmed to five categories, QuickBooks/templates removed
- 2026-09-27 `bak-intake-20260927` (66,847 B), `bak-merge`, `bak-options` — multi-step client-intake wizard added
- 2026-09-18→25 `bak-email`, `bak-favicon`, `bak-contact`, `bak-phoneparens`, `bak-nophone`, `bak-office`, `bak-addr`, `bak-addrave`, `bak-content` (47 K→50 K) — contact details, favicons, address iterations
- 2026-09-15 `faa8bf5 Initial deploy` + `index.html.bak-20260918` (47,732 B) — original site

## Troubleshooting Notes
1. **Never run a build.** There is no `package.json`; `npm install` / `npm run build` will fail and is never needed. Tailwind is the CDN runtime build — do not "fix" missing utilities by adding a `tailwind.config.js` or a PostCSS pipeline, and the Phase-5 "`@tailwind` in compiled CSS" check does not apply (there is no compiled CSS).
2. **Edit procedure that has worked 50+ times:** copy `index.html` to `/home/appuser/index.html.bak-<tag>-$(date +%Y%m%d)` **first**, apply the edit with a `python3 -I` helper written outside the web root (anchor-assert exactly once, `r+b` + `truncate` to preserve the inode/owner), delete the helper, then re-count tags and `curl` both pages. The strongest check available: strip the inserted blocks back out of the result (and revert any single-line replacements) and compare to the backup byte-for-byte — that proves the edit was pure. No nginx reload is required for content edits.
3. **Nothing with a `.py`, `.bak` or `.sh` extension may stay under `/home/appuser/app/`** — the whole directory is publicly served. Dotfiles and `.git` are already blocked by `location ~ /\.`.
4. **Served size is always ~99 B larger than the file on disk**, and `<script` counts one higher over the wire (12 vs 11 on disk today). nginx `sub_filter` injects the preview-reload `<script>` before `</body>` on every HTML response. This is the platform's iframe reload hook, not a corruption. Likewise `grep -c '<div'` counts *lines* and `grep -o | wc -l` counts *occurrences* — always compare the same method against a backup before concluding markup changed.
5. **There are SEVEN stacked style layers inside style block 1 (97–1125), and later wins.** Order: base (97–~630) → `ATTENTION & POLISH LAYER` (631–715) → `FLASH LAYER` (716–796) → `DYNAMIC LAYER` (797–894) → `HERO 3D LAYER` (895–964) → `HERO 3D DESKTOP` (965–999) → `REVIEWS 2.0` (1000–1084) → `DESKTOP MOTION` (1085–1124). The second `<style>` block is 2836–2852. The three motion scripts start at **3008** (DYNAMIC LAYER JS), **3107** (HERO 3D LAYER JS) and **3187** (DESKTOP MOTION JS). Consequences worth remembering:
   - `.hl` / `.hl-dark` are declared **twice** — flat bold color at 671–678 and the gradient-sweep override at 783–790. **Edit the FLASH LAYER copy**; editing only the earlier rule appears to do nothing.
   - `.hero-content` is declared **twice** — `animation: heroFadeIn … forwards; opacity: 0` at 196 and `animation: none; opacity: 1` at **634**. The 634 rule wins, so the hero has no filling animation and the DYNAMIC LAYER's inline `heroContent.style.transform/opacity` parallax works. If you ever delete line 634, the parallax silently stops (a `forwards`-filled animation outranks the inline style) and the children's `heroRise` stagger breaks.
   - `.hero-watermark` is animated twice — `wmFloat` (DYNAMIC LAYER) then `wmSpin3D` (HERO 3D LAYER, which wins), and HERO 3D DESKTOP only shortens its duration to 14 s.
   - `.hero-section` gained `overflow: hidden` in HERO 3D LAYER. Anything that must visually escape the hero box (a dropdown, a tooltip) will now be clipped.
   - Markup line numbers shift with every content edit — the 2026-11-19 credit card inserted 15 lines at 1424 and single lines at 1914 / 2145. Re-grep for anchors, never trust a remembered line number.
6. **The desktop depth fields are JS-injected, desktop-only, and must stay behind the text.** `.dk-layer` is `z-index: -1` inside a section that sets `isolation: isolate` — remove the `isolation` and the orbs sink behind the *page* background and vanish. `overflow: hidden` belongs on `.dk-layer`, never on the section (a section-level `overflow` would clip sticky/absolute children elsewhere). Both `.dk-layer` and `.hero-stage` are `pointer-events: none`.
7. **Desktop motion never runs on phones or with reduced motion.** The `DESKTOP MOTION JS` IIFE returns immediately when `matchMedia('(min-width: 769px)')` does not match or `prefers-reduced-motion: reduce` is set — so nothing is injected at all on mobile. The card tilt additionally checks `(hover: none)`. Keep all three guards if you extend it.
8. **Card tilt writes an inline `transform` and clears it on `pointerleave`.** Any CSS hover `transform` on `.service-card` / `.book-cover` is outranked while the pointer is inside the card; clearing the inline style hands control straight back to CSS. Do not add `!important` transforms to those cards — the tilt would freeze.
9. **Adding or removing a service category is a FIVE-point edit, not one.** The 2026-11-19 credit deploy is the template: (a) the card in `#services-grid-y5z6` (`sc<N>` + its `sc<N>a` icon wrapper and `sc<N>d` divider = 3 divs), (b) the `#services` heading word (`Five`/`Six Core Service Categories`), (c) the trust counter `data-id="ts2n"` `data-count`, (d) a `.cf-service` chip in the contact form, (e) an `.ik-service` chip in the intake wizard — both chips placed **before** their "Other" sibling. Miss (c) and the counter contradicts the grid; miss (d)/(e) and clients cannot select the service.
10. **Intake chips without a detail panel are legitimate.** `detailKeys()` looks up `ik-detail-<key>`; `ik22c` (`data-key="credit"`) has none on purpose, so it is skipped there and still flows through `services()` into the review screen and the lead email. Do not "fix" it by adding an empty panel.
11. **`.cf-service` / `.ik-service` greps overcount by design.** Raw `grep -o 'cf-service'` returns chip markup **plus** the JS selector strings `.cf-service:checked` (×1) and `.ik-service:checked` (×2). Count `class="cf-service"` / `class="ik-service"` for real chip totals — today **7** and **9**.
12. **`checklists/` is dead weight.** The live PDFs are in `resources/` (that is what `data-pdf="/resources/…"` points at); the `checklists/` copies have different md5s and are an October 3 leftover. Update `resources/` only — or you will "fix" a file nobody downloads.
13. **Git is not the safety net.** One commit, no remote, everything since is untracked, and root cannot run git here (`dubious ownership`; use `su appuser -c '…'`). Restore from the `.bak` files — `cp` onto `index.html` keeps the live inode 46335.
14. **SEO hosts are deliberately NOT the preview domain.** `canonical`, `og:url`, JSON-LD `@id`/`url`, `sitemap.xml` `<loc>` and `robots.txt` `Sitemap:` all point at `https://www.ash-financial.com/` on purpose; nginx compensates by sending `X-Robots-Tag: noindex, nofollow` for any `*.livemy.site` host. Do **not** rewrite these to the livemy domain (that would un-break a thing that is not broken and start indexing the preview).
15. **Lead flow is dual-path and asymmetric:** FormSubmit.co is the source of truth (awaited, errors surfaced in the UI); the Apps Script Sheet mirror is `no-cors`, `keepalive`, never awaited and never throws. A dead Sheet must never block a lead — keep that property if you touch `sendToSheet()`.
16. **`formsubmit.co` can take >10 s to answer from this container.** A 10-second probe returned `000`; at 25 s it returned 200 (and 405 on the POST-only ajax path for a GET). Do not report it as DEGRADED off a short-timeout probe.
17. **The chat service worker is scoped to `./` (`/chat/`) only** and passes every cross-origin request straight through — the Copilot iframe and Google Fonts are never cached. If chat UI changes do not appear, bump `CACHE = 'ash-ai-shell-vN'` in `chat/sw.js`; the main site has no service worker at all.
18. **Decorative layers must stay out of the way.** `.hero-section::after`, `.nav-wrap::after`, `.hero-stage`, `.hero-grid` and `.dk-layer` are all `pointer-events: none` and sit below the content (`z-index: 0` or `-1`; `.hero-content` is `z-index: 1`). The `#scroll-progress` bar and `.ripple` follow the same rule. If you add another overlay, keep both properties or hero buttons stop being clickable.
19. **Every animation added must also be disabled in a `prefers-reduced-motion` block.** There are five: FLASH LAYER (~793), DYNAMIC LAYER (~889), HERO 3D LAYER (~959), REVIEWS 2.0 (~1079) and DESKTOP MOTION (~1120). The DYNAMIC, HERO 3D and DESKTOP MOTION scripts also read `matchMedia('(prefers-reduced-motion: reduce)')` and skip their work entirely.
20. **The reveal layer can never hide content permanently.** `.rv` is added by JS only when `IntersectionObserver` exists, and a 4 s `setTimeout` force-adds `.shown` to every element regardless. Keep both guards if you extend the `sels` array — adding `.rv` in the HTML instead would hide content for JS-less visitors.
21. **Back-to-top lives on the LEFT by design.** `#ashChatBtn` (Ash AI bubble) owns the bottom-right corner; moving `#to-top` there would overlap it.
22. **`data-id` appears in CSS and JS, not just markup — 789 occurrences on disk today** (768 before the credit card, +21 from the new `sc6*`/`205c`/`ik22c` anchors). The real markup anchor count tracks this closely; everything above it is CSS attribute selectors (DYNAMIC LAYER, HERO 3D DESKTOP, REVIEWS 2.0) and JS strings. When auditing anchor preservation, subtract the `<style>` (97–1125, 2836–2852) and `<script>` ranges before comparing — a raw `grep -c` will look like markup changed when it did not. Likewise `trust-strip` greps as 5 but `id="trust-strip"` is 1, and `ashChatBtn` greps as 10 but `id="ashChatBtn"` is 1.
23. **`data-id` attributes are the platform's edit anchors** — preserve them byte-for-byte when rewriting markup, and give every new element its own unique one.
24. **Lucide icon names cannot be validated from the shell.** `data-lucide="gauge"` (card `sc6`) resolves at runtime from the unpkg CDN; a typo yields a silently missing icon with no server-side symptom. Check the name against the lucide icon list before writing it.
25. **If you ever add a backend**, use `$vh_fwd_proto` (already defined in `app.conf`) for `X-Forwarded-Proto` — never `$scheme`, which produces a 301 loop through Cloudflare. Keep nginx HTTP-only on :80; TLS belongs to the bastion.