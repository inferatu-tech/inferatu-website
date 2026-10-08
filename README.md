# inferatu-website

Static rebuild of [www.inferatu.com](https://www.inferatu.com), replacing the
WordPress 6.9 / Divi site. Hand-written `index.html` + `styles.css` + `assets/`,
no framework, no build step. Served by Cloudflare Workers static assets.

## Layout

| File | Purpose |
|---|---|
| `index.html` | The whole site (sections `#home`, `#what`, `#who`, `#where`, `#contact` — same anchors as the Divi site so old links work) |
| `styles.css` | Styles; values measured from the live Divi site (table below) |
| `assets/` | Logo (WebP + PNG), hero (WebP + JPEG), headshots (AVIF + JPEG), favicons 32/180/192/512 |
| `404.html` | Branded not-found page (`not_found_handling = "404-page"`) |
| `_redirects` | Legacy WordPress URLs → `/` or `/sitemap.xml` |
| `_headers` | Security headers, asset caching, `noindex` on `workers.dev` |
| `robots.txt`, `sitemap.xml` | Single-URL sitemap |
| `wrangler.toml` | Worker config (routes added after cutover — see below) |
| `.assetsignore` | Keeps config/repo files out of the public bundle |

## Local development

```bash
npx wrangler dev --port 8787 --persist-to /tmp/inferatu-wrangler-state
```

`--persist-to` outside the repo matters: with `[assets] directory = "."`,
wrangler otherwise writes `.wrangler/` inside the watched directory and
reload-loops. Syncthing writing into the folder also triggers reloads; if the
server stops answering, restart it.

Check: `curl -sI localhost:8787/` (200 + security headers), `/nope` (404),
`/feed/` (301 → `/`), `/wrangler.toml` (404).

`npx wrangler deploy --dry-run` validates the config. "Read N files" counts
`.git`/`.wrangler` before `.assetsignore` filtering; the 404s above prove the
filter works.

## Third-party services carried over

| Service | Value | Notes |
|---|---|---|
| Google tag (Site Kit) | `GT-TB78MQGW` | gtag snippet in `<head>` |
| Google Maps JavaScript API | key `AIzaSyAieMbw-pYylMDoq96Xw13EecFIjbENHM4` | Lazy-loaded via IntersectionObserver; JSON dark style, classic `Marker`. HTTP-referrer allowlist updated 2026-10-06 to include `*.workers.dev/*` and localhost dev origins. |
| Google Fonts | Josefin Sans 500/600/700, Open Sans 500/700 | Same families Divi loaded, trimmed to the weights used |
| HQ coordinates | 33.1010196, -117.0754548, zoom 9 | From the Divi map module |

## Phase 0 measurements (live Divi site, 2026-10-06)

Taken with `getBoundingClientRect()` / `getComputedStyle()` at 1366px wide.
Static version verified to match within 1–3px (page height 2665 vs 2661).

| Element | Live value |
|---|---|
| Top bar | 31px tall, Open Sans 12px/17px, white, ETmodules handset/envelope glyphs 12px; phone at x=30 (weight 600), email bold; pinned on scroll; centered on mobile |
| Main header | 80px tall, fixed, `#000`; **compacts to 54px** once `scrollY > 60` (`.et-fixed-header`), 0.4s ease-in-out |
| Logo | 43px tall at x=34 (260px wide); **29px tall (175px wide) when compact** |
| Nav links | Open Sans 14px 700 `rgba(255,255,255,.6)`, 26px apart, right edge x=1334 |
| Hero | 490px tall; `front-page.png` cover; padding 192px top/bottom |
| Hero title | Josefin Sans 96px/96px 700, `text-shadow 0 .1em .1em rgba(0,0,0,.4)` |
| Section padding | 54px top/bottom (`.et_pb_section` ≥1350px) |
| Row | 1080px wide, 27px padding top/bottom |
| H1 | Josefin Sans 30px/30px 700, padding-bottom 10px |
| Paragraph | Josefin Sans 16px, **line-height 23.8px** (Divi body `1.7em` of 14px, fixed), 500, justified, padding-bottom 16px |
| Team member | photo 150×150 r=5px, 30px gutter; h4 18px/18px 700 pb 10px; role 16px `#aaa`; member spacing 30px; LinkedIn icon 16px, margin-top 20px |
| Map | 440px tall (200px mobile) |
| Contact text | Josefin Sans 18px/23.8px 600, letter-spacing 1px, centered |
| Social follow | 32×32 r=3px `#007bb6`, 30px below text, 8px bottom margin |
| Footer | 54px tall; Open Sans 14px 500 `#666`; padding 15/5 + 10px on text |
| Mobile (390px) | header unchanged; hero 305px (title 48px, padding 192/55); section 50px, row 80% w/ 30px padding; map 200px; footer centered |

Deliberate departures from the live site:

- Body copy is 18px/28px (`--body-size`/`--body-lh`), up from Divi's
  16px/23.8px; Josefin Sans's small x-height made 16px read small.
  Approved 2026-10-08.

- The phone/email top bar is not pinned; it scrolls away and only the main
  header is sticky (compacting 80→54px, logo 43→29px, past 60px of scroll,
  as Divi does). Divi pins the top bar too, which costs 31px of viewport at
  every scroll position.
- Mobile shows the logo (Divi hid it) and left-aligns body text (justified
  text at 312px produced rivers).
- The X/Twitter "Follow" icon pointed at `#` on the live site; it is omitted.
  Add it back in `#contact .social-follow` when there is a real URL.
- Divi's "scroll to top" button is not reproduced.
- Images: hero PNG 382 KB → WebP 13 KB; logo 2560px PNG → 1040px WebP.

## State of the WordPress site at handover (2026-10-06)

- Host: `34.94.143.68` (nginx 1.31, PHP 8.2), WordPress 6.9.9, Divi 4.27.9 +
  Divi-Child.
- **TLS certificate expired 2026-08-07** (Let's Encrypt, CN=inferatu.com).
  Visitors currently get a browser warning. Cutover fixes this.
- Apex → www redirect is issued by WordPress (`X-Redirect-By: WordPress`);
  it must be replaced by a Cloudflare Redirect Rule.
- DNS is at **GoDaddy** (`ns27/ns28.domaincontrol.com`). Records:

  | Name | Type | Value |
  |---|---|---|
  | `@` | A | `34.94.143.68` |
  | `www` | CNAME | `inferatu.com` |
  | `@` | MX | `1 smtp.google.com` |
  | `@` | TXT | `v=spf1 include:_spf.google.com ~all` |
  | `@` | TXT | `google-site-verification=2OQJzFwT8y_NG1UmlcWM-KB0QDjl-G-9PCerCxYHRHk` |
  | `_dmarc` | TXT | `v=DMARC1; p=quarantine; sp=none; aspf=s; adkim=s;` |

  No DKIM selector records were found by probing common names; export the
  full GoDaddy zone before moving to catch any others.

## Deployment & cutover plan

### Phase 3 — Workers Builds

Workers Builds connects only to GitHub/GitLab, so this Origin repo needs a
**private GitHub mirror** (`git remote add github <ssh-url>`; push both:
`git push origin main && git push github main`).

Cloudflare → Workers & Pages → Create → Import a repository:

| Setting | Value |
|---|---|
| Build command | *(empty)* |
| Deploy command | `npx wrangler deploy` |
| Preview builds | on |
| Protect with Cloudflare Access | on |
| Root directory | `/` |

First deploy with **no `[[routes]]`** (as committed) so only the
`workers.dev` URL is created. Verify there: fonts, map (no
`RefererNotAllowedMapError` in the console), 404, redirects, headers,
ignored files.

### Phase 4 — Cutover (differs from intelligentprime.com: zone is not on Cloudflare yet)

1. **Add `inferatu.com` as a zone** in the Intelligent Prime Cloudflare
   account. Import/verify every record from the table above (MX, SPF, DMARC,
   site-verification). Set SSL/TLS to Full (strict), Always Use HTTPS on.
2. **Change nameservers at GoDaddy** to the two Cloudflare nameservers shown
   for the zone. Until this propagates, the site keeps serving from the old
   host (with its expired cert). Mail is unaffected as long as MX/TXT were
   copied correctly.
3. Create the zone **Redirect Rule** "Redirect from root to WWW"
   (`https://inferatu.com/*` → `https://www.inferatu.com/${1}`, 301,
   preserve query string).
4. Worker → Domains → **Add Domain** `www.inferatu.com`, Production only.
   Delete the `www` CNAME in the zone first (two tabs), then add.
5. Same for the apex: delete only the `A` record, then Add Domain. Expect up
   to ~40 minutes before the apex resolves; don't add placeholder records.
6. Commit to `wrangler.toml` in one change:

   ```toml
   workers_dev = false
   preview_urls = true

   [[routes]]
   pattern = "www.inferatu.com"
   custom_domain = true

   [[routes]]
   pattern = "inferatu.com"
   custom_domain = true
   ```

7. Run the skill's `verify-site.sh https://www.inferatu.com https://inferatu.com`,
   Google Rich Results Test, LinkedIn Post Inspector, and submit
   `/sitemap.xml` in Search Console.

### Phase 5 — Retire WordPress

Confirm a restorable backup exists, stop (don't delete) the instance for a
couple of weeks, then delete. Update the Maps key referrer list to drop any
entries that are no longer needed.

## Cutover log

- **2026-10-06** — Static site built and verified locally against the live
  Divi site. Pushed to Origin and to the GitHub mirror
  `inferatu-tech/inferatu-website`.
- **2026-10-06** — Workers Builds connected; first deploy succeeded
  (version `1c8cc070`, 17 assets, no config files leaked). Verified on
  `https://inferatu-website.red-bar-5885.workers.dev`: statuses, legacy
  redirects, security headers, `noindex` on workers.dev, fonts, map +
  marker (referrer allowlist OK), valid TLS.
- **2026-10-06 ~15:50 PT — Cutover.** Zone `inferatu.com` added to Cloudflare
  (`earl`/`mia.ns.cloudflare.com`), nameservers changed at GoDaddy, Redirect
  Rule created, `www` then apex attached as Worker custom domains. Verified
  from outside within minutes: `www` served by the Worker with a valid
  Let's Encrypt cert (the WordPress host's cert had expired 2026-08-07),
  apex 301 → www issued at the edge (no `X-Redirect-By`), MX/SPF/DMARC/
  site-verification intact. The apex resolved immediately — no 40-minute
  publish gap this time. `[[routes]]` + `workers_dev = false` +
  `preview_urls = true` committed afterwards.
- **2026-10-06 ~16:10 PT — Post-cutover.** `verify-site.sh` all green;
  `workers.dev` confirmed off. Rich Results Test, LinkedIn Post Inspector
  and Search Console sitemap submission done. Two lessons:
  - Search Console showed "Couldn't fetch" with an empty *Last read* right
    after submitting `/sitemap.xml`, although Googlebot-UA fetches returned
    200 `application/xml`. That status is the pre-first-crawl placeholder;
    it cleared on its own. Use *URL Inspection → Test live URL* on the
    sitemap to confirm immediately instead of waiting.
  - **Always Use HTTPS had not been enabled**; plain-HTTP requests returned
    200 and the apex served a duplicate of the site over HTTP because the
    Redirect Rule only matches `https://`. Found by curling `http://` URLs
    after cutover — add that to the verification. Enabled; chain is now
    `http://inferatu.com/x` → `https://inferatu.com/x` → `https://www.inferatu.com/x`.
  - WordPress GCP VM at `34.94.143.68` shut down. Backups: the UpdraftPlus
    set on Backblaze B2 plus a GCP archival disk snapshot taken before
    shutdown.
- **2026-10-08** — Body copy raised to 18px/28px. WordPress VM **deleted**
  and its static IP released; the B2 backup and GCP snapshot remain the only
  copies of the old site. Maps key referrer list still includes the
  `localhost`/`127.0.0.1` dev origins by choice; prune when local work ends.
  Migration closed.
