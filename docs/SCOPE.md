# Scope: Feed Reader + Read-It-Later

Status: scoping draft, pre-design. Last updated 2026-09-14.
Companion: [LANDSCAPE.md](LANDSCAPE.md) (competitive survey, open-source
bases, Kindle opportunity, UX thesis).
Decisions taken so far: free and open source, no fees; no server we operate;
on-device processing and iCloud sync in the user's own quota; no telemetry;
SwiftUI client first; phased plan as in §8; Windows and X deferred past v1.

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

### Product principles (non-negotiable)

These sit above the feature list and decide the architecture in §6.

| Principle | Consequence |
|---|---|
| Free, with no paywalled basics | No subscription, no in-app purchase, no feed-count cap, no gated OPML import. Reeder's free tier stops at 10 feeds and withholds OPML import; that is the behaviour we exist to avoid. |
| Open source | Permissive licence (MIT recommended, NetNewsWire's precedent). Anyone can audit, build, and fork. |
| No server we operate | Nothing to pay for, nothing to shut down, nothing that dies with the maintainer. Our only recurring cost is the Apple Developer Program. |
| The user's own iCloud is the sync layer | Data lives in the user's private CloudKit database, under their quota, readable by no one else including us. |
| On-device processing | Fetching, extraction, EPUB building, image processing all run on the user's devices. |
| No telemetry, no calling home | No analytics SDK, no third-party crash reporter, no update pings to a server of ours. Network traffic goes only to the user's sources, iCloud, and the user's own mail provider. |
| Data is never locked in | OPML, EPUB, JSON and CSV export at all times. Deleting the app leaves nothing behind but the user's own iCloud records, which they can wipe. |

## 3. Feature scope

### Core (must have)

- Subscribe to sources: RSS/Atom/JSON Feed, YouTube channels/playlists,
  podcast feeds, Reddit subreddits/users, Bluesky and Mastodon accounts,
  newsletters via a personal inbound email address.
- Scheduled editions with triage UI (read / save / skip), keyboard-driven on
  desktop, swipe-driven on phone.
- Read-later queue: save from inside the app, from the iOS/macOS share sheet,
  from a Safari extension (Mac and iOS, which can talk to the app without a
  server), and via Shortcuts.
- Clean article extraction with full-text archiving (survives the source
  changing or dying), offline reading, reading position sync.
- Export to Kindle: single article on demand, and an automatic daily digest,
  sent from the user's own mail account or via the Kindle app share sheet.
- Cross-device sync of subscriptions, read state, queue, and positions via the
  user's iCloud.
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
- Any server we operate, user accounts, analytics, or telemetry.
- Windows, Android and Linux clients. iCloud-only sync makes them second-class;
  see §7 for the later path.
- X as a source (no free API).

## 4. Source adapters: feasibility

Ranked by confidence. "Risk" is the chance the integration breaks or becomes
costly within two years.

| Source | Mechanism | Auth / cost | Risk | Notes |
|---|---|---|---|---|
| RSS / Atom / JSON Feed | Standard fetch + parse, conditional GET, WebSub where offered | None | Low | Mature libraries in every language. Feed discovery from a site URL is a solved problem. |
| YouTube | Undocumented but long-lived RSS: `youtube.com/feeds/videos.xml?channel_id=UC…` (also `playlist_id=`) | None | Low-Med | Resolving a handle or URL to a channel ID requires scraping the channel page or one call to the YouTube Data API (free quota is ample). Feed gives title, thumbnail, description, no duration. |
| Podcasts | Ordinary RSS with `<enclosure>`; iTunes Search API for discovery | None | Low | Playback is the expensive part, not the feed. |
| Newsletters | Read the user's own mailbox over IMAP on device, filtered by label or sender; convert HTML email to an item. Also accept feeds from email-to-feed services the user chooses (Kill the Newsletter). | Mailbox access granted by the user; no cost | Low | feeeed ships the on-device Gmail approach today. Sensitive permission; see §6 and §9. |
| Mastodon | Built-in RSS on every profile (`/@user.rss`); public API needs no key for public data | None | Low | |
| Bluesky | Public AT Protocol API (no key for public data); RSS available per profile | None | Low | Good substitute for X for many accounts. |
| Reddit | `reddit.com/r/<sub>/.rss` still works unauthenticated; OAuth Data API is free for non-commercial use at 100 requests/min | None for RSS; OAuth app registration for API | **Medium-High** | Unauthenticated RSS is rate limited by user agent and has been throttled before. Reddit announced in Aug 2026 that new public Data API requests will be restricted and third-party apps pushed to its Devvit platform. Plan for RSS-only, degrade gracefully, and treat the API as a bonus. Commercial use is effectively off the table (~$12k/month minimum). |
| X / Twitter | No RSS. API is pay-per-use only for new developers: about $0.005 per post read, no free tier | Paid | **High** | Following 50 accounts at 20 posts/day each is ~$150/month in reads alone. Third-party mirrors (Nitter-style) are unreliable and against X's terms. Recommendation: defer X, support Bluesky/Mastodon, and offer "paste an X link to save it" which works via the public embed endpoint for single posts. |
| Arbitrary web pages | Scrape-to-feed (RSS-Bridge style selectors or change detection) | None | Medium | Useful but a maintenance sink. Later phase. |

## 5. The two hard integrations

### 5.1 Kindle export

**There is no public Send to Kindle API.** Everything goes through one of
Amazon's user-facing channels. Feasible routes, best first:

1. **Email to `name@kindle.com` from the user's own mail account.** Amazon
   accepts EPUB attachments (50 MB limit, up to 25 documents per message)
   from addresses on the user's approved-sender list. Since April 2026 the
   approved entry must be a full address. Most Kindle owners already have
   their own email address approved, so the app sends from that account over
   SMTP (iCloud Mail, Gmail, Fastmail and others support app-specific
   passwords; credentials stay in the Keychain). Readwise and Instapaper do
   this from their servers; we do it from the device. Enables the automatic
   daily digest, which is the feature that most directly serves the
   anti-doomscrolling goal. Automation is reliable from a Mac; on iPhone it
   runs overnight in a background processing task, best effort, with a
   one-tap notification as fallback.
2. **Share to the Kindle app.** The Kindle app on iOS/iPadOS/macOS exposes a
   Send to Kindle share extension. Our app generates the EPUB locally and
   hands it to the share sheet. Zero setup, works in the MVP, manual.
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
- **Delivery tracking.** Email is fire-and-forget; we can confirm the mail
  server accepted the message, not that Amazon delivered it. If the user has
  granted mailbox access for newsletters (§6), we can also watch for Amazon's
  failure and verification emails and surface them inline, which no
  competitor does. Set expectations in the UI.
- **Highlights back from Kindle** are only available via the
  `My Clippings.txt` file (USB) or the Kindle notebook export from the
  device's share menu (email). Optional, later.

Other e-readers, for the design doc's benefit:

- **Kobo** replaced Pocket with **Instapaper** as its only read-later
  integration (2025). There is no third-party hook, and routing through
  Instapaper's API would mean a third-party account, which our principles
  rule out. Kobo gets EPUB/KePub export via Files and USB. Its on-device
  highlights database is readable when mounted, so a Mac can import them.
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

## 6. Architecture shape: serverless, iCloud-first

The product principles rule out a hosted backend. Everything runs on the
user's devices; the user's private CloudKit database is the only shared
state, and it lives in their own iCloud quota.

An earlier draft assumed a server for four reasons. Each has a serverless
answer, with a trade-off we accept:

| Need | Serverless approach | Trade-off |
|---|---|---|
| Scheduled fetching | Fetch on app open (with conditional GET, a hundred feeds refresh in seconds, so the edition assembles instantly). iOS Background App Refresh is opportunistic on top. The Mac app fetches on a real timer and pushes results through iCloud, so an always-on Mac acts as the household's fetcher. | iPhone-only users may see the edition assemble when they open the app rather than find it waiting. The edition model absorbs this: the schedule gates *presentation*, not fetching. |
| Newsletters | Read the user's own mailbox over IMAP on device (iCloud Mail, Gmail, Fastmail; app-specific password or OAuth), filter by label or sender, convert to items. Also accept feeds from any email-to-feed service the user picks (Kill the Newsletter). feeeed already ships the Gmail approach. | Asks the user for mailbox access, which is a sensitive permission. Everything stays on device. |
| Email to Kindle | Send from the user's own mail account over SMTP (see §5.1), or hand the EPUB to the Kindle app's share extension. | Automated nightly delivery is reliable from a Mac; on iPhone it is best effort. |
| Sync | CloudKit private database through CKSyncEngine (iOS 17 / macOS 14 and later). NetNewsWire proved iCloud sync works for a reader; CKSyncEngine removes most of the pain it had with the older APIs. | Apple-only. Windows cannot be a first-class client (§7). |
| Extraction | swift-readability (pure Swift port of Mozilla Readability) plus SwiftSoup, in-process, also inside the share extension. Fallback for JavaScript-rendered pages: load in a hidden WKWebView, then extract. | On-device quality must be validated against a corpus in Phase 0. |
| EPUB build | A small Swift EPUB 3 writer of our own (zip via ZIPFoundation; XHTML, OPF, nav document). Image pipeline with vImage or Core Image: resize to device width, grayscale, dither, compress. | No Swift EPUB writer exists, so we write roughly a few hundred lines. The format is simple and Apple's image frameworks are excellent for this. |
| Archive storage | Extracted HTML and downscaled images stored as assets in the private database, cached locally. | Counts against the user's iCloud quota (5 GB on the free tier). Downscaled articles run tens to a couple of hundred kilobytes; provide retention settings and show usage. |

Component shape:

```
iPhone / iPad / Mac app (one SwiftUI codebase)
 ├─ Fetch engine: RSS/Atom/JSON Feed, YouTube RSS, Reddit RSS, Bluesky & Mastodon APIs, IMAP newsletters
 ├─ Edition builder: schedule-gated assembly, triage state, rules
 ├─ Extractor + archiver (on device)
 ├─ EPUB builder + e-ink image pipeline (on device)
 ├─ Delivery: Kindle app share extension | SMTP from the user's account | Files export
 ├─ Share extension, Safari extension, widgets, Shortcuts actions
 └─ Sync: CKSyncEngine ↔ iCloud private database (user's quota)
```

Network traffic leaves the device only toward the user's sources, iCloud,
and the user's own mail provider. There is no analytics SDK and no crash
reporter beyond Apple's opt-in system diagnostics.

Cost to the maintainer: the Apple Developer Program at $99/year. Code hosting
and CI on GitHub are free for public repositories, and Xcode Cloud has a free
tier. Nothing else.

Honest limits of this model, to be stated plainly in the README:

- Windows, Android and Linux are not in v1. The later path is a static web
  client using CloudKit JS against the same private database, hosted for free
  on GitHub Pages and signed in with the user's Apple ID. Real work, but no
  server.
- iPhone-only automation is best effort. Pair with a Mac for a reliable
  nightly edition.
- A Kobo two-way path through Instapaper's API is out. Kobo gets file export.

Two things the model does better than a server, worth saying:

- Reddit and YouTube are fetched from each user's own device, so rate limits
  apply per person rather than to one shared server IP that Reddit can block.
- There is no account to create, no login screen, and nothing for us to leak.

## 7. Platform strategy

Decided: **one SwiftUI codebase for iPhone, iPad and Mac.** The
save-from-anywhere share extension, Safari extension, widgets, Shortcuts and
the Apple News hand-off are all native features, and iCloud sync is
Apple-only by construction.

Minimum OS: whatever ships CKSyncEngine comfortably, iOS 17 / macOS 14 today,
likely iOS 18 / macOS 15 by launch. Decide in Phase 0.

Windows is deferred, not abandoned. The path is a CloudKit JS web client
reading the same private database, hosted statically. Revisit after Phase 2
if there is demand.

## 8. Phased plan and sizing

Sizing is for one experienced developer working on this seriously but not
full-time. S ≈ up to a week, M ≈ 2–3 weeks, L ≈ 4–6 weeks, XL ≈ longer or
open-ended. Treat these as relative, not commitments.

### Phase 0: Decide and de-risk (M)

- Lock the remaining decisions in §9 (licence, minimum OS, mailbox access
  timing, retention default).
- Repository, CI (GitHub Actions or Xcode Cloud), TestFlight, privacy
  statement.
- Spike: CKSyncEngine data model for sources, items, editions, queue and
  read state. Measure sync latency between an iPhone and a Mac.
- Spike: Swift EPUB writer producing a two-article file that passes epubcheck,
  arrives via the Kindle app share sheet, and arrives via SMTP from an iCloud
  Mail account. Judge it on a real Kindle.
- Spike: swift-readability against a 30-page corpus (news, blogs, Substack,
  docs pages, JavaScript-heavy sites). Decide when to fall back to WKWebView.
- Spike: measure Background App Refresh and background processing task
  behaviour on a real iPhone over a week.

### Phase 1: MVP (L–XL)

- Sources: RSS/Atom/JSON Feed, YouTube channel and playlist feeds, podcast
  feeds (as items), feed discovery from a URL, OPML import and export.
- Editions on a schedule, triage view (read now, tonight, never) with
  keyboard on Mac and swipe on iPhone, "already seen" dimming, end state.
- Read-later queue, on-device extraction and archive, reader view, offline.
- Share extension to save a URL.
- iCloud sync across iPhone, iPad and Mac.
- Kindle: EPUB via the Kindle app share sheet; on-demand single-article send
  over SMTP from the user's account.
- Mac timer-based fetching.

Exit criterion: you use it daily instead of the apps it replaces.

### Phase 2: The edition on paper (L)

- Nightly digest edition: sections, cover, flat-safe table of contents,
  rolling replacement of yesterday's edition, e-ink image pipeline.
- Automated send from Mac; best-effort overnight send from iPhone with a
  one-tap fallback notification. Delivery status from SMTP acceptance.
- Newsletters via IMAP, and with it detection of Amazon's failure emails.
- Bluesky and Mastodon adapters.
- Filters and mute rules, search over archived text, widgets.

### Phase 3: Round trip and extras (M–L)

- Reddit adapter with graceful degradation.
- Apple News+ hand-off and `apple.news` link unwrapping.
- Highlights and notes; import from a mounted Kindle's clippings file on the
  Mac; Kobo EPUB/KePub export and highlight import.
- Podcast "send to app" and, optionally, a basic player.
- Shortcuts actions and a Safari extension.

### Later / only if justified

- Static CloudKit JS web client for Windows and other platforms.
- Authenticated fetch for the user's own web subscriptions.
- Scrape-to-feed for sites without RSS.
- X, only if its API becomes free for personal use.

## 9. Decisions

### Taken

- Free, open source, no fees, no paywalled basics.
- No server we operate. iCloud private database is the sync layer.
- On-device processing. No telemetry.
- One SwiftUI codebase for iPhone, iPad and Mac. Windows deferred.
- Phased plan as in §8.
- X dropped from v1.

### Still open before the design doc

1. **Licence.** MIT recommended: NetNewsWire's precedent, App Store friendly,
   lets others build on it. GPL-family licences have known friction with App
   Store terms.
2. **Minimum OS version** (iOS 17/macOS 14 vs 18/15). Newer buys better
   CKSyncEngine and SwiftData behaviour; older reaches more devices.
3. **Mailbox access timing.** Newsletters and Kindle-failure detection need
   IMAP access. Phase 2 as planned, or later, given how sensitive the
   permission is?
4. **Archive retention default.** Keep full text and images for saved items
   forever, or for a window, given it lives in the user's iCloud quota.
5. **Podcasts:** items with hand-off only, or a basic player in Phase 3?
6. **Funding.** None, or an optional GitHub Sponsors link with no in-app
   mention. Either is compatible with the principles.
7. **Distribution.** App Store (needed for iOS anyway) plus a notarised Mac
   download from GitHub releases, or App Store only.
8. **Name.**

## 10. Key risks

| Risk | Impact | Mitigation |
|---|---|---|
| iOS background execution is stingy | Editions and nightly sends on iPhone are late or skipped | Fetch-on-open makes staleness invisible; Mac as the reliable scheduler; one-tap fallback notification; measure in Phase 0 |
| iCloud sync conflicts or latency | Read state flickers, duplicates | CKSyncEngine with last-writer-wins per field, small records, conflict tests in Phase 0; NetNewsWire's experience as a guide |
| Users on the 5 GB iCloud tier run out of space | Sync stops | Aggressive image downscaling, retention defaults, visible usage meter |
| Reddit throttles unauthenticated RSS | Lose a wanted source | RSS-first design, per-source health indicator, per-device fetching spreads load |
| Kindle email conversion drops content or images | Bad first impression of the headline feature | Phase 0 spike, epubcheck, test corpus, conservative image sizing |
| Amazon changes Send to Kindle rules again | Delivery breaks | Stay on the sanctioned email path; keep the Kindle app share sheet and file export as fallbacks |
| On-device extraction quality | Constant low-grade annoyance | Corpus bake-off, WKWebView fallback, per-site overrides, "view original" escape hatch |
| Storing mail credentials | Trust and review risk | Keychain only, app-specific passwords, clear in-app explanation, no server ever sees them |
| YouTube removes the undocumented feed | Lose YouTube | Adapter isolated; fallback to the Data API's free quota with a user-supplied key |
| Scope for one developer | Nothing ships | Apple-only, strict phase exits, Phase 1 exit is "you use it daily" |
| Apple News+ expectations | Disappointment if "integration" is read as "save News+ articles" | Name it "Open in News" and document the limit |

## 11. Comparable products to study

- **Readwise Reader**: feeds + read-later + newsletters + Kindle digest. The
  closest commercial analog; study its triage and digest flows.
- **NetNewsWire** (MIT, free, iCloud sync): the closest analog to our
  model and the codebase to learn most from. See LANDSCAPE.md §1a for why we
  build alongside it rather than contribute editions and Kindle to it.
- **feeeed** (free, no account, on-device, Gmail newsletters): proof that the
  serverless model works commercially, though closed source.
- **Omnivore** (open source, AGPL, discontinued): SwiftUI reader view and
  highlights worth studying; licence makes it study-only.
- **Wallabag**: self-hosted read-later with EPUB export and Kindle email.
- **Feedbin**: hosted RSS with newsletter addresses and YouTube support.
- **Miniflux / FreshRSS**: mature feed fetchers; study their handling of feed edge cases (redirects, encodings, dead feeds, conditional GET).
- **Reeder, Unread, lire**: reference for Apple-native reader UI quality.
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
