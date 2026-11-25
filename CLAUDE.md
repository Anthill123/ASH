# Project: ASH Financial — static marketing site + Ash AI chat PWA

## Overview
ASH Financial (`www.ash-financial.com`) is a single-page financial-services marketing site: one hand-written `index.html` (~185 KB, 719 `data-id` anchors) plus a separate `/chat/` PWA that embeds a Microsoft Copilot Studio agent ("Ash AI Assistant"). **There is no build step, no Node process, no Docker, no backend** — nginx serves `/home/appuser/app` as a plain directory on port 80, and all dynamic behaviour (lead forms, intake wizard, animations) is inline vanilla JS talking to third-party endpoints (FormSubmit.co + a Google Apps Script web app). Editing the site = editing `index.html` in place; nginx needs no reload, only a browser refresh (HTML is served `no-cache, must-revalidate`).

Live preview: `https://vh-prod-main-5e2352-762b3e42.livemy.site` (returns 200, 0 redirects). Canonical production host declared in the markup: `https://www.ash-financial.com/`.

## Stack
| Layer | What is actually used |
|---|---|
| Markup | Hand-authored HTML5, single file `index.html` + `chat/index.html` |
| CSS | Inline `<style>` blocks (2 of them) + **Tailwind via CDN** `https://cdn.tailwindcss.com` (runtime JIT, **no PostCSS/tailwind.config.js, nothing compiled**) |
| Icons | `lucide` via `https://unpkg.com/lucide@latest` (runtime) |
| Fonts | Google Fonts — Inter (300–700), Montserrat (400–900) |
| JS | Vanilla ES5-style inline scripts (8 `<script>` blocks in `index.html`). No bundler, no framework |
| PWA | `/chat/sw.js` (cache `ash-ai-shell-v3`), `/chat/manifest.json`, root `site.webmanifest` |
| Server | nginx 1.18.0 (Ubuntu), static root, no proxy_pass anywhere |
| Host runtime | Ubuntu 24.04 LXC; node v22.22.2 + npm 10.9.7 and python3/docker/git installed but **unused by this project** |
| Package manager | none — no `package.json`, no `node_modules`, no lockfile |

Brand tokens: `--brand: #C8102E` (ASH red), `--brand-dark: #a00d24`, dark-section highlight gold `#e0b951`, ink `#0a0a0a`.

## File Structure
```
/home/appuser/app/                 <- nginx root, owned appuser:appuser
├── index.html            185,266 B  THE site — all markup, CSS, JS in one file
├── chat/                            Ash AI Assistant PWA (own scope, own icons)
│   ├── index.html          9,352 B  shell + full-bleed iframe to Copilot Studio
│   ├── sw.js               1,527 B  service worker, scope ./ only (never touches the main site)
│   ├── manifest.json                standalone PWA manifest
│   ├── logo.png, favicon.svg, favicon.ico, icon-192.png, icon-512.png, apple-touch-icon.png
├── resources/                       ** LIVE PDFs — these are the ones linked from index.html **
│   ├── bookkeeping-checklist.pdf           md5 aa2d06dd…
│   ├── new-business-startup-checklist.pdf  md5 2d7446c5…
│   ├── tax-planning-checklist.pdf          md5 34d35269…
│   └── tax-preparation-checklist.pdf       md5 15132c41…
├── checklists/                      ** STALE duplicate set (Oct 3), NOT referenced anywhere **
├── assets/logo-ashai.png, logo-ashai-light.png
├── favicon.ico / favicon.svg / favicon-96.png / icon-192.png / icon-512.png / apple-touch-icon.png
├── site.webmanifest, robots.txt, sitemap.xml
├── .gitignore
└── .git/                            local-only repo, no remote, single commit "faa8bf5 Initial deploy"
```
Backups live **outside** the web root in `/home/appuser/` as `index.html.bak-<tag>-<YYYYMMDD>` (50+ files, Sep 2026 → Nov 2026). Never put a `.bak` or a helper `.py` inside `/home/appuser/app/` — it would be publicly served.

## Entry Points
| URL | Served from | Notes |
|---|---|---|
| `/` | `/home/appuser/app/index.html` | 200, 185,365 B over the wire (99 B larger than on disk — nginx `sub_filter` appends the preview-reload script) |
| `/chat/` | `/home/appuser/app/chat/index.html` | 200, 9,451 B; iframes Copilot Studio |
| `/resources/*.pdf` | static files | delivered after a lead form is submitted |
| `/sitemap.xml`, `/robots.txt`, `/site.webmanifest` | static | all point at `www.ash-financial.com`, not the preview host |
| anything else | `try_files $uri $uri/ $uri.html /index.html` | SPA-style fallback to the homepage |

## Scripts
**There are no npm scripts — no `package.json` exists.** Nothing is built, bundled, minified or transpiled. The deploy "build" is: write `index.html` → nginx serves it on the next request.

Useful real commands instead:
```bash
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/        # 200
curl -s -o /dev/null -w '%{http_code}\n' http://localhost/chat/   # 200
sudo nginx -t && sudo systemctl reload nginx                      # only after editing app.conf
cp /home/appuser/app/index.html /home/appuser/index.html.bak-<tag>-$(date +%Y%m%d)   # ALWAYS before an edit
```

## Key Dependencies
All dependencies are **runtime CDN / third-party services** — there is nothing installed on the VM.

| Dependency | Purpose | Probe result (2026-11-16) |
|---|---|---|
| `cdn.tailwindcss.com` | Tailwind utility classes, compiled in the browser | 302 (alive) |
| `unpkg.com/lucide@latest` | icon set, `lucide.createIcons()` at load | 302 (alive) |
| `fonts.googleapis.com` / `fonts.gstatic.com` | Inter + Montserrat | alive |
| `formsubmit.co/ajax/contact@ash-financial.com` | **source of truth** for every lead → email | 200 at root, 405 on the ajax path for GET (normal — POST-only) |
| `script.google.com/a/macros/ash-financial.com/s/AKfycbz…/exec` | Google Sheets mirror of leads (fire-and-forget, `mode:'no-cors'`) | 302 (alive) |
| `copilotstudio.microsoft.com` | the Ash AI chat agent iframe | 200 (alive) |
| `www.amazon.com/dp/...` | book purchase links | outbound links only |

## Routes / Pages
Single page, anchor-navigated. Sections in `index.html` (id → purpose):

| `#id` | `data-id` | Heading / purpose |
|---|---|---|
| `#home` | 37 | Hero — "Building Wealth. Creating Legacy." |
| `#trust-strip` | ts0 | Trust badges strip |
| `#value-prop` | vp0 | Value proposition |
| `#about` | ab0 | Our Story |
| `#mission` | 47 | Mission |
| `#philosophy` | ph0 | Philosophy |
| `#services` | 67 | Five Core Service Categories |
| `#why-us` | wc0 | Why Choose ASH Financial |
| `#how-we-serve` | bm0 | How ASH Financial Serves Clients |
| `#who-we-help` | wh0 | Who We Help |
| `#testimonials` | tm0 | What Clients Say |
| `#books` | 108 | The Money Blueprint Series (4 Amazon links: `B0HJQ62RYF`, `B0HLW1RWTG`, `B0HNM56KZH`, `B0GYK6L3P7`) |
| `#resources` | cr0 | Free Resources & Checklists (4 gated PDFs) |
| `#contact` | 168 | Contact form + `#intake` multi-step client-intake wizard |

Sticky mobile section nav: `<nav id="mobile-tabs-mt0" class="mtabs">` with `.mtab[data-scroll]` → Home / About / Mission / Services / Reviews / Books (+ more), kept in sync with scroll position by `centerTab()` / `onScroll()`.

Second page: `/chat/` — header + install button + `<iframe id="chatFrame">` pointing at Copilot Studio environment `e7efbfde-51ed-e3f5-93d9-56ca968a800f`, bot `cr8f7_ashai_pWNnCq`.

## Configuration

### nginx — `/etc/nginx/sites-available/app.conf` (symlinked into `sites-enabled/`, this is the FULL active config)
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
Verified: `nginx -T | grep -E '^\s*(listen 443|ssl_certificate)'` → **empty** (no TLS inside the container, no `/tmp/` cert paths). No `proxy_pass` exists, so `$vh_fwd_proto` is currently unused but is pre-declared for any future backend. Config backup: `/home/appuser/app.conf.bak-seo-20261015`.

### Environment variables
**None.** No `.env`, `.env.production` or `.env.local` anywhere in the project. Every secret-looking value (Apps Script URL, Copilot bot id) is intentionally public and baked into the HTML.

### Build output
None. The served directory *is* the source directory (`/home/appuser/app`).

### systemd units
Only platform/base units — this project adds none:
`nginx.service` (enabled, active), `vibhost-agent.service` (VibHost gRPC agent, port 50051), `docker.service`, `containerd.service`, `fail2ban.service` (1 jail: sshd), `ssh.service`, `cron`, `snapd`, `systemd-resolved`, `unattended-upgrades`.

## Runtime State (verified 2026-11-16 17:19 UTC)
- **Uptime:** 15 days, 4:36. Load 0.32 / 0.43 / 0.46.
- **Listening ports:** `:80` nginx (pid 442/451), `:22` sshd, `*:50051` vh-agent, `127.0.0.1:45537` containerd, `127.0.0.53:53` systemd-resolved. **Nothing else — no app process.**
- **Docker:** daemon running, **zero containers** (`docker ps` empty).
- **No node / python / pm2 / gunicorn process belongs to this project.**
- **Disk:** `/` and `/home` on `/dev/loop8` — 1.2 T total, 74 G used, 1.1 T free (7 %).
- **Memory:** 32,095 MB total, ~10.5 G used, ~21 G available, no swap.
- **Health:** `curl localhost/` → 200 / 185,365 B; `curl localhost/chat/` → 200 / 9,451 B; `curl -H 'Host: vh-prod-main-5e2352-762b3e42.livemy.site' -H 'X-Forwarded-Proto: https' http://localhost/` → `200 OK` with `Cache-Control: no-cache, must-revalidate` and `X-Robots-Tag: noindex, nofollow`. Public URL → 200, 0 redirects.
- **git:** repo at `/home/appuser/app/.git`, no remote, one commit `faa8bf5 Initial deploy`; `index.html` shows as modified and everything added after the initial commit (chat/, assets/, resources/, icons, sitemap…) is untracked. Nothing is committed on edit — **git is not the backup mechanism here; the `.bak` files are.** Running git as root hits `detected dubious ownership` — use `su appuser -c 'cd /home/appuser/app && git …'`.

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
| Browser storage key | `localStorage['ashIntakeV1']` (intake wizard draft), `localStorage['ashAiHintSeen']` (iOS install hint) | — |

## Deploy History

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
2. **Edit procedure that has worked 50+ times:** copy `index.html` to `/home/appuser/index.html.bak-<tag>-$(date +%Y%m%d)` **first**, apply the edit with a `python3 -I` helper written outside the web root (anchor-assert exactly once, `r+b` + `truncate` to preserve the inode/owner), delete the helper, then re-count tags and `curl` both pages. No nginx reload required for content edits.
3. **Nothing with a `.py`, `.bak` or `.sh` extension may stay under `/home/appuser/app/`** — the whole directory is publicly served. Dotfiles and `.git` are already blocked by `location ~ /\.`.
4. **Served size is always ~99 B larger than the file on disk.** nginx `sub_filter` injects the preview-reload `<script>` before `</body>` on every HTML response. This is the platform's iframe reload hook, not a corruption.
5. **`checklists/` is dead weight.** The live PDFs are in `resources/` (that is what `data-pdf="/resources/…"` points at); the `checklists/` copies have different md5s and are an October 3 leftover. Update `resources/` only — or you will "fix" a file nobody downloads.
6. **Git is not the safety net.** One commit, no remote, everything since is untracked, and root cannot run git here (`dubious ownership`; use `su appuser -c '…'`). Restore from the `.bak` files.
7. **SEO hosts are deliberately NOT the preview domain.** `canonical`, `og:url`, JSON-LD `@id`/`url`, `sitemap.xml` `<loc>` and `robots.txt` `Sitemap:` all point at `https://www.ash-financial.com/` on purpose; nginx compensates by sending `X-Robots-Tag: noindex, nofollow` for any `*.livemy.site` host. Do **not** rewrite these to the livemy domain (that would un-break a thing that is not broken and start indexing the preview).
8. **Lead flow is dual-path and asymmetric:** FormSubmit.co is the source of truth (awaited, errors surfaced in the UI); the Apps Script Sheet mirror is `no-cors`, `keepalive`, never awaited and never throws. A dead Sheet must never block a lead — keep that property if you touch `sendToSheet()`.
9. **`formsubmit.co` can take >10 s to answer from this container.** A 10-second probe returned `000`; at 25 s it returned 200 (and 405 on the POST-only ajax path for a GET). Do not report it as DEGRADED off a short-timeout probe.
10. **The chat service worker is scoped to `./` (`/chat/`) only** and passes every cross-origin request straight through — the Copilot iframe and Google Fonts are never cached. If chat UI changes do not appear, bump `CACHE = 'ash-ai-shell-vN'` in `chat/sw.js`; the main site has no service worker at all.
11. **If you ever add a backend**, use `$vh_fwd_proto` (already defined in `app.conf`) for `X-Forwarded-Proto` — never `$scheme`, which produces a 301 loop through Cloudflare. Keep nginx HTTP-only on :80; TLS belongs to the bastion.
12. `data-id` attributes (719 of them) are the platform's edit anchors — preserve them byte-for-byte when rewriting markup.
