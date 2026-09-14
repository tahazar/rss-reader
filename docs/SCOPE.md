# Scope: Feed Reader + Read-It-Later

Status: scoping draft, pre-design. Last updated 2026-09-14.
Companion: [LANDSCAPE.md](LANDSCAPE.md) (competitive survey, open-source
bases, Kindle opportunity, UX thesis).
Decisions taken so far: phased plan as in §8; SwiftUI client first (§7 option A).

This document answers "what would this take?" before a formal design doc. It
covers the product idea, what is and is not technically feasible for each
integration, platform strategy options, a phased plan with rough sizing, and
the decisions that must be made before design starts.

---

## 1. The idea in one paragraph

A calm reader that pulls everything you follow (RSS/Atom, YouTube channels,
podcasts, Reddit, social, newsletters) into one finite inbox, lets you save
anything from anywhere into a read-later queue, and pushes that queue to the
surface where you actually read without distraction: an e-reader. Available
on iPhone, Mac and Windows. The anti-doomscrolling goal is a design constraint,
not a feature: the app should have an end state ("you're done") and should
never manufacture reasons to keep scrolling.

## 2. Anti-doomscrolling design principles

These decide what we deliberately do *not* build, and should carry into the
design doc as requirements.

| Principle | What it means in the product |
|---|---|
| Finite editions, not a stream | Feeds are fetched on a schedule you pick (e.g. 07:00 and 18:00), delivered as a numbered "edition" with a known length. No pull-to-refresh, no live updating. |
| A visible end state | When the edition is read or triaged, the screen says so and stops. No "you might also like". |
| Triage first, read second | The inbox is for deciding: read now, save for later, skip. Reading happens in the queue or on the e-reader. |
| No unread counts, no badges by default | Optional cap ("12+") if the user wants it. |
| No algorithmic ranking, no recommendations | Chronological within the sources you chose. |
| Long-form leaves the phone | The default path for anything over a few minutes is "goes to Kindle tonight", not "read on the phone now". |
| Social sources are read as posts, not as a timeline | Reddit/Bluesky/Mastodon come in as batched items in the edition, never as an infinite feed view. |
| No autoplay, no video inline by default | YouTube items are a thumbnail and a description; tapping opens the player deliberately. |

## 3. Feature scope

### Core (must have)

- Subscribe to sources: RSS/Atom/JSON Feed, YouTube channels/playlists,
  podcast feeds, Reddit subreddits/users, Bluesky and Mastodon accounts,
  newsletters via a personal inbound email address.
- Scheduled editions with triage UI (read / save / skip), keyboard-driven on
  desktop, swipe-driven on phone.
- Read-later queue: save from inside the app, from the iOS/macOS share sheet,
  from a browser extension, and by emailing a link.
- Clean article extraction with full-text archiving (survives the source
  changing or dying), offline reading, reading position sync.
- Export to Kindle: single article on demand, and an automatic daily digest.
- Cross-device sync of subscriptions, read state, queue, and positions.
- OPML import/export (feeds) and CSV/JSON export (queue) so data is never
  locked in.

### Should have

- Tags/folders for sources and for saved items.
- Highlights and notes on saved articles, synced back from Kindle where the
  Kindle "My Clippings" export allows it.
- Apple News+ hand-off (see §5.4) on iOS/Mac.
- Search across archived full text.
- Rules: "anything from this source over 2,000 words goes straight to the
  queue", "mute items matching X".
- Podcast handling: at minimum a per-episode "send to my podcast app" button;
  optionally a basic built-in player.

### Explicitly out of scope (v1)

- Paywall circumvention. We will support the user's *own* subscriptions
  (authenticated fetch, Apple News+ hand-off) and nothing that defeats
  paywalls the user has not paid for.
- Social posting, commenting, or replying. Read-only everywhere.
- Full podcast client (queue management, chapters, playback speed, CarPlay).
  Hand off to Overcast/Apple Podcasts and revisit later.
- Recommendations, trending, discovery beyond a plain search.
- Android and Linux clients (web client covers them if needed).

## 4. Source adapters: feasibility

Ranked by confidence. "Risk" is the chance the integration breaks or becomes
costly within two years.

| Source | Mechanism | Auth / cost | Risk | Notes |
|---|---|---|---|---|
| RSS / Atom / JSON Feed | Standard fetch + parse, conditional GET, WebSub where offered | None | Low | Mature libraries in every language. Feed discovery from a site URL is a solved problem. |
| YouTube | Undocumented but long-lived RSS: `youtube.com/feeds/videos.xml?channel_id=UC…` (also `playlist_id=`) | None | Low-Med | Resolving a handle or URL to a channel ID requires scraping the channel page or one call to the YouTube Data API (free quota is ample). Feed gives title, thumbnail, description, no duration. |
| Podcasts | Ordinary RSS with `<enclosure>`; iTunes Search API for discovery | None | Low | Playback is the expensive part, not the feed. |
| Newsletters | Give each user an inbound address (`u-abc@in.yourapp.example`); receive via a mail provider webhook, convert HTML email to an item | Mail service (~free at personal volume) | Low | Requires a hosted backend. Same pattern as Feedbin, Kill the Newsletter, Readwise. |
| Mastodon | Built-in RSS on every profile (`/@user.rss`); public API needs no key for public data | None | Low | |
| Bluesky | Public AT Protocol API (no key for public data); RSS available per profile | None | Low | Good substitute for X for many accounts. |
| Reddit | `reddit.com/r/<sub>/.rss` still works unauthenticated; OAuth Data API is free for non-commercial use at 100 requests/min | None for RSS; OAuth app registration for API | **Medium-High** | Unauthenticated RSS is rate limited by user agent and has been throttled before. Reddit announced in Aug 2026 that new public Data API requests will be restricted and third-party apps pushed to its Devvit platform. Plan for RSS-only, degrade gracefully, and treat the API as a bonus. Commercial use is effectively off the table (~$12k/month minimum). |
| X / Twitter | No RSS. API is pay-per-use only for new developers: about $0.005 per post read, no free tier | Paid | **High** | Following 50 accounts at 20 posts/day each is ~$150/month in reads alone. Third-party mirrors (Nitter-style) are unreliable and against X's terms. Recommendation: defer X, support Bluesky/Mastodon, and offer "paste an X link to save it" which works via the public embed endpoint for single posts. |
| Arbitrary web pages | Scrape-to-feed (RSS-Bridge style selectors or change detection) | None | Medium | Useful but a maintenance sink. Later phase. |

## 5. The two hard integrations

### 5.1 Kindle export

**There is no public Send to Kindle API.** Everything goes through one of
Amazon's user-facing channels. Feasible routes, best first:

1. **Email to `name@kindle.com` (server-side).** Amazon accepts EPUB
   attachments (50 MB limit, up to 25 documents per message) from addresses
   on the user's approved-sender list and converts them server-side. Our
   backend generates an EPUB and sends it from a per-user or shared address
   the user has whitelisted once in Amazon's settings. This is what Readwise,
   Instapaper and Wallabag do. Enables the automatic daily digest, which is
   the feature that most directly serves the anti-doomscrolling goal.
2. **Share to the Kindle iOS app (on-device, no server).** The Kindle app on
   iOS/iPadOS/macOS exposes a Send to Kindle share extension. Our app
   generates the EPUB locally and hands it to the share sheet. Zero
   infrastructure, works in the MVP, but manual and Apple-only.
3. **Send to Kindle web/desktop/browser extension, USB, Calibre.** Manual.
   We just make sure our EPUB export is good so these work.

What we must build regardless of route:

- **EPUB generation** from extracted article HTML: sanitize HTML to the EPUB
  subset, download and inline images (resize; Kindle has a per-file size
  cap and slow e-ink image decoding), generate cover, table of contents, and
  metadata. A digest is one EPUB with one chapter per article. Kindle's old
  "periodical" format (MOBI with sections) is deprecated; a well-structured
  EPUB is the right target.
- **Extraction quality.** Kindle output is unforgiving: a botched extraction
  (missing paragraphs, comment sections included, code blocks mangled) is
  far more annoying on e-ink than on a phone. Budget real time for an
  extraction test corpus.
- **Delivery tracking.** Email is fire-and-forget; we can only confirm
  "sent", not "arrived". Set expectations in the UI.
- **Highlights back from Kindle** are only available via the
  `My Clippings.txt` file (USB) or the Kindle notebook export from the
  device's share menu (email). Optional, later.

Other e-readers, for the design doc's benefit:

- **Kobo** replaced Pocket with **Instapaper** as its only read-later
  integration (2025). There is no third-party hook. Options: route via the
  Instapaper API (requires requesting Full API access), or rely on
  EPUB/KePub sideloading. Low priority unless you own a Kobo.
- **reMarkable, Boox, others**: EPUB export plus their own sync apps. Boox
  runs Android, so a web client covers it.

### 5.2 Apple News+

**Apple News has no read API.** The Apple News API exists only for
publishers to push and manage their own articles; it cannot fetch News+
content, and News+ articles are DRM'd inside the News app. Nothing lets a
third-party app extract News+ text, so News+ articles cannot be saved to the
queue as text or sent to Kindle.

What *is* feasible, and cheap:

- **Hand-off.** Since iOS 14 / macOS Big Sur, News+ subscribers with "Open
  Web Links in News" enabled get publisher links opened in the News app,
  paywall-free. We can maintain a list of News+ publisher domains and show an
  "Open in Apple News" action on those items, or simply open the publisher
  URL and let the OS redirect. Apple-only, but this is exactly what the user
  asked for: reading their subscriptions without hitting the paywall.
- **Resolve `apple.news` links.** Shared News links are short URLs that
  redirect to a page containing the canonical publisher URL. We can unwrap
  those so a saved News link still shows title, source and original URL in
  the queue even if the body must be read in News.
- **Authenticated fetch for the user's own web subscriptions** (not News+):
  an in-app browser session where the user logs in to, say, a newspaper
  they pay for, and extraction runs against the logged-in page. Legitimate,
  common in read-later apps, but fragile and higher effort. Phase 3 at the
  earliest.

Honest verdict: Apple News+ integration is a hand-off button and link
unwrapping. It is not a source adapter and not a route to Kindle. Worth
doing because it is small; not worth designing the product around.

## 6. Architecture shape

The requirements force a **hosted backend** (self-hostable, but running
somewhere) rather than a purely local app:

- Scheduled fetching cannot rely on iOS background execution.
- Newsletter inbound email needs a server that receives mail.
- Email-to-Kindle needs a sender with a stable, whitelisted address.
- Sync across Apple and Windows rules out an iCloud-only design.

Proposed components:

```
clients (iOS/Mac, Windows/web, share ext, browser ext)
        │  HTTPS JSON API + sync
        ▼
backend: API + auth ── job scheduler ── fetchers (RSS, YouTube, Reddit, AT Proto, Mastodon)
                     ├─ inbound mail webhook → newsletter items
                     ├─ extractor (readability-class) + archiver (HTML, images)
                     ├─ EPUB builder → outbound mail (Kindle) / download
                     └─ storage: Postgres or SQLite + object store for archives
```

Build vs. adopt for the backend:

- **Adopt** an existing self-hosted feed server (Miniflux, FreshRSS) via its
  API for fetching, and build only the read-later/Kindle service alongside.
  Saves the fetcher and feed-edge-case work. Costs you two systems to run and
  a data model split.
- **Build** one backend with off-the-shelf libraries for parsing (gofeed,
  feedparser, rss-parser) and extraction (Mozilla Readability, Defuddle,
  trafilatura). More work up front, one coherent model, editions and rules
  become straightforward.

Recommendation: build, deriving the backend core from Miniflux (Apache-2.0,
Go) rather than from zero, because editions/triage/queue/Kindle are the
product and the feed fetcher is the smallest piece. Study Omnivore's
open-source codebase (AGPL-3.0, iOS/Mac/web/newsletters/feeds) before
designing; it is the closest existing template even though the service shut
down, but its licence makes it study-only unless we also ship AGPL. See
[LANDSCAPE.md](LANDSCAPE.md) §3–4 for the full build-vs-reuse survey.

## 7. Platform strategy

The user's priority order is iOS, Mac, Windows, with e-reader as the reading
surface. Three realistic options:

| Option | Codebases | Strengths | Weaknesses |
|---|---|---|---|
| **A. SwiftUI for iOS + macOS, web app for Windows** | 2 (Swift, web) | Best share-sheet, widgets, Shortcuts, Apple News hand-off, offline. One Swift codebase covers iPhone, iPad, Mac. Web client also serves Android/Linux/Boox. | Windows gets a browser-grade experience (installable PWA or Tauri wrapper). |
| B. Flutter (or Compose Multiplatform) everywhere | 1 (+ small native share extension) | One UI codebase for all three targets. | Share extension and Apple integrations still need native code; desktop keyboard/menu polish is weaker; less "native" feel on Mac. |
| C. Web-first (PWA) + thin wrappers | 1 | Fastest to something usable on all platforms. | iOS PWA share-target and offline are limited; a native share extension is still needed for save-from-anywhere. |

Recommendation: **A**. The save-from-anywhere flow and Apple News hand-off
are native features, and iOS+Mac share one SwiftUI codebase. The web client
is needed anyway for Windows and for the browser extension, so the marginal
cost of "Windows via web" is low. Revisit a native Windows app only if the
web client proves inadequate.

## 8. Phased plan and sizing

Sizing is for one experienced developer working on this seriously but not
full-time. S ≈ up to a week, M ≈ 2–3 weeks, L ≈ 4–6 weeks, XL ≈ longer or
open-ended. Treat these as relative, not commitments.

### Phase 0: Decide and de-risk (S–M)

- Lock decisions in §9.
- Spike: EPUB generation from five messy real articles, sent to a real Kindle
  by email and by the Kindle app share sheet. Judge output quality on e-ink.
- Spike: extraction library bake-off on a 30-article corpus (news, blogs,
  Substack, docs pages, paywalled-but-subscribed).
- Spike: Reddit `.rss` behaviour under realistic polling from a server IP.

### Phase 1: MVP, Apple-first (L–XL)

- Backend: accounts, subscriptions, scheduled fetch for RSS/Atom/JSON
  Feed/YouTube/podcast feeds, editions, items, queue, read state, sync API.
- Extraction + archive on save.
- iOS/Mac app: onboarding, add source, edition triage view, queue, reader,
  settings for schedule.
- Share extension: save URL to queue.
- Kindle: local EPUB + share to Kindle app; on-demand single-article email.
- OPML import.

Exit criterion: you use it daily instead of the apps it replaces.

### Phase 2: Digest, Windows, newsletters (L)

- Daily Kindle digest with per-user schedule and per-source rules.
- Web client (Windows and everywhere else), browser extension for save.
- Newsletter inbound address.
- Bluesky and Mastodon adapters.
- Search over archived text.

### Phase 3: Social and Apple extras (M–L)

- Reddit adapter with graceful degradation.
- Apple News+ hand-off and `apple.news` unwrapping.
- Highlights/notes; Kindle clippings import.
- Podcast "send to app" and optional basic player.

### Later / only if justified

- X via paid API (cost decision, see §4).
- Kobo via Instapaper API.
- Authenticated fetch for the user's own web subscriptions.
- Scrape-to-feed for sites without RSS.

## 9. Decisions needed before the design doc

1. **Personal tool or product?** Affects Reddit terms (non-commercial only),
   X API cost, App Store review, and how much multi-user/auth work is needed.
   A personal or small-group tool is a much smaller project.
2. **Hosting.** Self-hosted single binary you run yourself, or a hosted
   service? This drives the storage choice (SQLite vs Postgres) and how much
   ops work is in scope.
3. ~~**Platform option** from §7 (recommendation: A).~~ **Decided: A, SwiftUI first.**
4. **Podcasts:** feed-only with hand-off, or a player in v1?
5. **Kindle delivery default:** email from a shared app address (one-time
   whitelist, simplest) or per-user sender addresses (more robust against
   Amazon rate limits and abuse, more setup).
6. **X:** drop for v1, or budget for pay-per-use?
7. **Data retention:** archive full text and images for every saved item
   forever, or for a window? Storage cost and privacy stance.

## 10. Key risks

| Risk | Impact | Mitigation |
|---|---|---|
| Reddit blocks unauthenticated RSS or restricts API further | Lose a wanted source | RSS-first design, per-source health indicator, user-supplied OAuth app as fallback |
| Kindle email conversion silently drops content or images | Bad first impression of the headline feature | Phase 0 spike, EPUB validation (epubcheck), test corpus, conservative image sizing |
| Extraction quality on real-world pages | Constant low-grade annoyance | Library bake-off, site-specific overrides, "view original" escape hatch |
| YouTube changes or removes the undocumented feed | Lose YouTube | Adapter isolated; fallback to Data API within free quota |
| Three-platform scope for one developer | Nothing ships | Apple-first, web for Windows, strict phase exits |
| Apple News+ expectations | Disappointment if "integration" is read as "save News+ articles" | Name it "Open in News" in the UI and document the limit |

## 11. Comparable products to study

- **Readwise Reader**: feeds + read-later + newsletters + Kindle digest. The
  closest commercial analog; study its triage and digest flows.
- **Omnivore** (open source, AGPL, discontinued): iOS/Mac/web, newsletters,
  feeds. Best open codebase to learn from; licence makes it study-only.
- **Wallabag**: self-hosted read-later with EPUB export and Kindle email.
- **Feedbin**: hosted RSS with newsletter addresses and YouTube support.
- **Miniflux / FreshRSS**: self-hosted feed servers with clean APIs.
- **NetNewsWire / Reeder**: reference for Apple-native reader UI quality.
- **Instapaper**: Kindle digest since 2010; now Kobo's official partner.

## 12. Sources checked while scoping

- Send to Kindle: EPUB by email accepted since 2022, 50 MB limit
  (https://stacktobook.com/blog/send-to-kindle-complete-guide,
  https://toolkit.bot/blog/epub-send-to-kindle)
- Apple News API is publish/manage only
  (https://developer.apple.com/documentation/applenewsapi)
- News+ links open in the News app for subscribers since iOS 14
  (https://www.macrumors.com/2020/08/10/apple-news-plus-ios-14-web-links/)
- X API pay-per-use, no free tier for new developers
  (https://postproxy.dev/blog/x-api-pricing-2026/,
  https://www.socialcrawl.dev/blog/x-twitter-api-2026)
- Reddit RSS and API status
  (https://www.socialcrawl.dev/blog/reddit-data-api-2026,
  https://prowlo.com/blog/reddit-data-api)
- YouTube channel RSS still works
  (https://blog.pesky.moe/posts/2024-11-24-yt-rss/)
- Kobo replaced Pocket with Instapaper
  (https://www.kobo.com/news/rakuten-kobo-announces-plans-for-instapaper-integration-continuing-commitment-to-seamless-read-it-later-experience)
