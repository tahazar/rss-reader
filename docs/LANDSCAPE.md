# Landscape: readers, open-source bases, and where to innovate

Status: research synthesis, pre-design. Last updated 2026-09-14.
Companion to [SCOPE.md](SCOPE.md). Decisions already taken: free and open
source, no server we operate, on-device processing with iCloud sync, no
telemetry, SwiftUI client first, phased plan as scoped.

This document answers three questions:

1. What does a modern feed reader have to have? (§1, §2)
2. What can we build on instead of writing from scratch? (§3, §4)
3. Where is the room to contribute something new, given that feed lists are
   a solved problem? (§5, §6, §7)

Research was done against live sources on 2026-09-14. URLs are collected in
§9. Where a claim rests on a search excerpt rather than a page we could open,
it is marked "(unverified)".

---

## 1. How the market is split

Three clusters, with almost no overlap:

| Cluster | Examples | Competes on | Typical weakness |
|---|---|---|---|
| Power-user services | Inoreader, Feedly, Readwise Reader, Folo | Rules, filters, AI summaries, newsletters, breadth of sources | Subscription fatigue, AI upsell, "reading as work" |
| Indie Apple clients | NetNewsWire, Reeder Classic, Unread, lire, Tapestry | Typography, gestures, one-time or cheap pricing, breadth of sync services | No filters/rules, iCloud sync slowness, no newsletters, no server |
| Finite / calm readers (2025–26) | Current, Today, JorvikDailyNews, Doomscroll News, digest services (Readless, Digest, Meco) | Abolishing unread counts, bounding the day's reading | Young, thin feature sets, mostly single-platform |

Two facts about this split matter for us:

- **The "finite" category is real and growing.** Current (Feb 2026) launched
  on "a river, not an inbox"; Today (Oct 2025) on "a single daily digest, no
  endless scroll"; JorvikDailyNews builds a today-only newspaper. Our
  anti-doomscrolling premise is not contrarian any more. It is a category we
  would enter, so the differentiation has to come from somewhere else.
- **Removing read state entirely backfires.** The new Reeder dropped
  read/unread in favour of a synced timeline position and drew sustained
  complaints ("can't tell what I've already read", no widget, iCloud only).
  Users want no *counts* and no *guilt*, but they still want a visible
  "already seen" signal. That is a precise UX requirement.

### 1a. The obvious question: why not just use or extend NetNewsWire?

NetNewsWire already is free, MIT-licensed, telemetry-free and iCloud-synced,
which is exactly our model. Three reasons to build alongside it rather than
inside it:

- **Product shape.** NetNewsWire is a classic unread-inbox reader. Editions,
  triage-first, no unread counts and "long-form leaves the phone" invert its
  core model; that is a fork in philosophy, not a feature request.
- **Read-later and Kindle are out of its scope.** Its maintainers have kept
  it a feed reader on purpose, and its most-requested features (filtering,
  custom smart feeds) have been open for years.
- **Architecture.** It is AppKit and UIKit with a large legacy surface.
  Starting in SwiftUI with CKSyncEngine is cheaper than retrofitting.

What we do take from it: its MIT-licensed Swift packages for feed parsing and
article storage, its CloudKit sync experience, and its stance on privacy.
Contributing fixes upstream where we find them is the right neighbourly
posture.

## 2. Table stakes for a modern reader

Features a reviewer or a switcher will look for on day one. Phase column maps
to the plan in SCOPE.md §8.

| Feature | Expected because | Our phase |
|---|---|---|
| OPML import and export | Universal; Today even subscribes to a remote OPML URL | 1 |
| Folders or tags for sources | Universal | 1 |
| Full-text extraction and reader view, offline | lire and Unread set the bar; NetNewsWire's is considered unreliable | 1 |
| Read-later / save from share sheet | Reeder, Unread, Readwise, Matter, Instapaper | 1 |
| Starred / bookmarked items | Universal | 1 |
| Dark mode, typography controls, themes | Universal on Apple | 1 |
| Keyboard-driven triage on Mac (j/k, space, s, etc.) | NetNewsWire, Unread, Reeder, Inoreader, Readwise | 1 |
| "Already read" signal without unread counts | Reeder backlash (see §1) | 1 |
| Search across saved and archived text | Feedbin, Readwise, Matter, Inoreader | 2 |
| Filters and mute rules | Missing from all indie Apple clients; NetNewsWire's most-requested feature | 2 |
| Newsletter inbox address | Feedbin (unlimited), Readwise, Inoreader, Feedly Pro+; absent from every indie client | 2 |
| YouTube and podcast items handled sensibly | Reeder's unified media player, Feedbin's YouTube chapters | 1 (items), 3 (player) |
| Social sources (Mastodon, Bluesky, Reddit) | Reeder, Tapestry, feeeed, Folo | 2–3 |
| Widgets | Complaint when missing (new Reeder) | 2 |
| Highlights and notes | Readwise, Matter, Instapaper, Omnivore | 3 |
| Sync across devices with no visible lag | iCloud readers are criticised for slowness; account-based services are not | 1 (own backend) |
| Exposes or speaks a standard sync API | NetNewsWire, Reeder Classic, Unread, lire all speak Google Reader / Fever / Feedbin APIs | 1 (see §4) |

Features we can skip without being judged for it: AI summaries and chat
(Inoreader, Feedly, Folo, Readwise all have them and all draw "upsell"
complaints), social layers and creator tipping (Folo), text-to-speech
(Matter, Instapaper).

Pricing observed, for context (we have decided on free with no paywalled
basics): free open source
(NetNewsWire, feeeed), one-time $5–10 (Reeder Classic, lire, Current), cheap
subscriptions $10–20/yr (Reeder, Tapestry), mid $30–90/yr (Unread, Instapaper,
Feedbin, Feedly, Inoreader), premium ~$120/yr (Readwise). Instapaper doubled
its price in 2025 and put Kindle delivery behind Premium in 2026; both drew
strong backlash, which says the audience is price-sensitive and Kindle
delivery is something they expected to be included. Reeder's free tier stops
at ten feeds and withholds OPML import, so a switcher cannot even bring their
subscriptions over without paying; that is the pattern we are positioned
against.

## 3. Open-source bases: what exists

### 3.1 Apple client

There is **no permissively licensed SwiftUI reader to fork**. The options:

| Project | License | Verdict |
|---|---|---|
| NetNewsWire | MIT, Swift, AppKit/UIKit (not SwiftUI), active | Study and lift pieces: Google Reader API client, feed refresh scheduling, article store, HTML sanitising. Do not fork as a base; the UI layer fights SwiftUI. |
| Omnivore (apple/) | **AGPL-3.0** (whole monorepo; SCOPE.md previously said Apache, which was wrong) | Study only. Best reference for SwiftUI reader view and highlight UX. |
| Readeck iOS | MIT, SwiftUI, small (41 stars) | Read in an afternoon for structure; too small to fork. |
| FeedFlow | Apache-2.0, Kotlin Multiplatform core with SwiftUI shell | Study SwiftUI screens; not a Swift base. |
| wallabag iOS | MIT, archived March 2026 | Skip. |

Swift libraries worth vendoring: **FeedKit** (MIT, active, RSS/Atom/JSON Feed
parsing for on-device previews), **SwiftSoup** (MIT, HTML DOM),
**lake-of-fire/swift-readability** (BSD-3, pure Swift port of Mozilla
Readability passing its fixture suite, young so pin a commit) for
share-extension previews and offline fallback. No production-grade Swift EPUB
*writer* exists; EPUB generation belongs on the server.

### 3.2 Backend

| Project | License | Stack | Speaks | Kindle/EPUB | Verdict |
|---|---|---|---|---|---|
| **Miniflux** | Apache-2.0 | Go + Postgres | Own REST, Google Reader API subset, Fever | No | **Best fork or heavy-borrow candidate.** Small, proven fetcher and scheduler, permissive licence, and existing Apple clients already speak its protocols. |
| FreshRSS | AGPL-3.0 | PHP | Google Reader, Fever | No | Use via API only. |
| Feedbin | MIT | Rails + sidecars | Own REST | **Yes, email EPUB since 2014** | Study its Kindle pipeline and API design; self-hosting officially discouraged. |
| Readeck | AGPL-3.0 | Go, single binary, SQLite | REST (OpenAPI) | **Yes, EPUB export and email-to-Kindle**, OPDS | Closest single project to our read-later half. Study; AGPL blocks embedding. |
| Wallabag | MIT | PHP/Symfony | REST | EPUB/PDF export; community Kindle mailer | Study its `graby` extraction and site configs. |
| Shiori | MIT | Go | REST | No (its org maintains go-epub) | Permissive Go read-later reference; parts vendorable. |
| Omnivore backend | AGPL-3.0 | Node/GraphQL/Puppeteer | GraphQL | No | Study the save/parse queue. |
| Folo | AGPL clients | TS | Backend is **closed**, self-hosting declined | No | UI ideas only. |
| Karakeep, Linkding, NewsBlur, TT-RSS | various | | | | Study only or skip. |

Sidecars to run as separate services, licence-isolated behind HTTP: **RSSHub**
(AGPL, feeds for thousands of sites), **RSS-Bridge** (Unlicense),
**Kill the Newsletter** (MIT, the inbound-email-to-feed pattern to copy).

### 3.3 Extraction

| Library | License | Runtime | Notes |
|---|---|---|---|
| trafilatura | Apache-2.0 | Python | Top of the ScrapingHub benchmark (F1 ≈ 0.945) |
| go-trafilatura | Apache-2.0 | Go | Active; falls back to go-readability and go-domdistiller; weaker on images |
| Defuddle | MIT | JS | Very active, multi-pass, normalises footnotes/code/math, emits Markdown; anecdotally beats Postlight |
| Mozilla Readability | Apache-2.0 | JS (+ Swift ports) | Baseline |
| go-readability | MIT | Go | **Archived Dec 2025** |
| Postlight Parser | Apache-2.0 | Node | Dormant; avoid |
| FiveFilters site-config | — | XPath rules | The per-site rule set is the useful part |

### 3.4 EPUB and Kindle tooling

- **go-epub** (go-shiori fork, MIT): maintained EPUB 3 writer with NCX,
  embeds images/CSS/fonts. The natural choice for a Go backend.
- **pandoc** (GPL, CLI): fine for HTML → EPUB; needs metadata for Send to
  Kindle to accept it.
- **Calibre** `ebook-convert` and its 1,000+ **news recipes** (GPL): the only
  tool that produces true newspaper-style periodicals with sections. Heavy;
  only ever as an optional sidecar container.
- **RSSPub** (MIT, Rust, active): feeds + read-later saves → daily EPUB
  "newspaper" → SMTP to Kindle, grayscale image optimisation. Small and
  readable; the closest thing to our Kindle pipeline that exists.
- **KindleEar** (MIT, Python, active): multi-user feeds + Calibre recipes →
  scheduled Kindle email, e-ink web reader, AI summaries. Powerful but
  hobbyist UX.
- **inkfeed** (AGPL, Go, 2026): Kindle-browser RSS reader with EPUB email.
- **kindle-send**, **digest-delivery**, **wallabag-kindle-consumer**: small
  URL-to-Kindle mailers.

## 4. Recommendation: reuse, fork, write

The serverless decision removes the entire backend column from §3.2. Those
projects remain useful to read, not to run. What is left is Swift.

**Write fresh: the SwiftUI app.** No permissively licensed SwiftUI reader
exists, and UX is the differentiator anyway.

**Vendor (Swift packages):**

- **FeedKit** (MIT) for RSS, Atom and JSON Feed parsing. NetNewsWire's own
  parser package (MIT) is the alternative if FeedKit's edge-case handling
  disappoints in Phase 0.
- **SwiftSoup** (MIT) for HTML handling and sanitising.
- **lake-of-fire/swift-readability** (BSD-3) for on-device extraction. Young;
  pin a commit and contribute fixes upstream.
- **ZIPFoundation** (MIT) under our own small EPUB 3 writer.
- Apple frameworks for the rest: CKSyncEngine for sync, vImage or Core Image
  for e-ink image processing, BackgroundTasks for scheduling, MailCore-class
  library or a minimal SMTP client for sending from the user's account.

**Lift code and lessons from (MIT):** NetNewsWire's CloudKit account,
article database and feed-refresh scheduling; Shiori's readable-archive
approach; Feedbin's Kindle pipeline for what an EPUB digest should contain.

**Study only (AGPL or closed):** Omnivore's SwiftUI reader view and
highlights, Readeck's EPUB and mailer behaviour, Folo and Reeder for UI
ideas, KindleEar and RSSPub for digest structure and image handling, Calibre
recipes for how a real periodical is sectioned.

**Not applicable any more:** Miniflux, FreshRSS, RSSHub, RSS-Bridge, Kill the
Newsletter as things we run. Users may still point the app at a public
RSSHub or Kill the Newsletter instance if they choose; that is their call,
not our infrastructure.

**Sequencing.** Without a backend, the dogfooding trick of using NetNewsWire
as an interim client goes away. Phase 0 spikes carry that de-risking load
instead: the CKSyncEngine data model, the EPUB writer on a real Kindle, and
the extraction corpus, before the main UI is built.

**Licence.** MIT for our own code, matching every Swift dependency above and
NetNewsWire. This keeps AGPL code strictly study-only and avoids the App
Store friction the GPL family has had.

## 5. The Kindle opportunity, examined

The hunch that Kindle is where the opportunity lies survives contact with the
research, with sharper edges.

### 5.1 What the incumbents do

| Product | Kindle delivery | Weakness |
|---|---|---|
| Readwise Reader | Daily/weekly EPUB digest at a fixed time, per-item send | $9.99/mo; flat digest with no sections; highlights on sent documents cannot come back (Readwise's own docs say so) |
| Instapaper | Digest since 2010; **Premium-only since early 2026** because email delivery is "resource intensive" | Paywalling drew loud backlash; flat list |
| FiveFilters Push to Kindle | $34.99/yr, per-article | Duplicated/missing/off-centre images, truncated paywalled sites |
| Amazon's own extension and share sheet | Free | Sign-in loops, stuck previews, retained ads/headers, bloated files |
| Calibre recipes | Free | Only tool with real sections and periodical structure; needs an always-on machine, silently fails on sender approval |
| KindleEar, RSSPub, KTool, Readivio, RSS to Kindle, Any2K | Various | Hobby or small SaaS; flat digests; none track delivery or round-trip |

### 5.2 Constraints we must design around

- Email is the only sanctioned automation path. 50 MB per message, 25
  attachments, sender must be on the approved list, and **since April 2026 the
  approved entry must be a full address**, not a domain. Reverse-engineered
  clients of the desktop app or read.amazon.com exist but are terms-of-service
  grey and break without notice.
- The Kindle "Go To" menu is driven by the EPUB nav document, in reading
  order. Nested TOCs flatten in practice. Cover must be marked or Amazon uses
  the first image.
- **Highlights and notes on personal documents never reach Amazon's cloud.**
  They sync between the user's own devices, but do not appear in the web
  notebook and "Export Notes" is greyed out for them. The only routes back are
  `My Clippings.txt` over USB or the Kindle desktop app. No third party can
  read them server-side.
- The Kindle browser (firmware 5.16.4+, WebKit, has an "Article Mode") can
  render a low-JS server-side UI, but it is slow and flashy. Viable for
  triage, not for reading. The Kindle-browser reader category (Reabble,
  kinss) is effectively abandoned since 2020–2025.
- Kindle Scribe's Active Canvas works on reflowable EPUBs sent via Send to
  Kindle. Ink exports leave only by emailed link.
- Kobo is the *open* e-reader: Instapaper integration is API-based, and
  highlights live in a readable on-device SQLite database with KePub. Real
  round-trip is possible there, not on Kindle.

### 5.3 Gaps nobody fills

1. **Delivery you can trust.** Every service fire-and-forgets. None confirms
   arrival, retries, or explains the two silent failure modes (sender not
   approved, confirmation email unanswered).
2. **A real edition, not a flat list.** Only Calibre produces sections, a
   front page and per-section TOC. Every SaaS digest is a stack of articles.
3. **E-ink asset quality.** Dithering and contrast for images, width-fitting,
   table-to-list and code-block reflow are unsolved in read-later tools and
   are the most common formatting complaints.
4. **Library hygiene.** One document per article clutters the Kindle library;
   duplicates persist; nobody manages a rolling edition that replaces
   yesterday's.
5. **Round-trip.** Impossible via Amazon's cloud, but nobody has productised
   the clippings-file path or made Kobo's open route a flagship.
6. **Transcripts on e-ink.** Readwise shows demand for YouTube and podcast
   transcripts; nobody renders them for a Kindle with chapters and speaker
   turns.
7. **A Kindle-browser triage companion.** Zero competition, modest value.

### 5.4 Ranked opportunities

1. **Morning edition with sections, delivered from the user's own account.**
   Calibre-quality periodical structure, rolling replacement of yesterday's
   edition, and a per-edition delivery timeline in the app (mail server
   accepted; Amazon failure email detected when mailbox access is granted).
   Free, with no server in the loop, it directly answers Instapaper's
   paywall and the SaaS crowd's flat digests. This is the headline feature.
2. **E-ink typography and asset pipeline.** Per-source CSS profiles,
   dithered and contrast-boosted images sized to the target device, table and
   code reflow. Cheap relative to its visibility; every competitor is bad at
   it.
3. **Best-effort highlight round-trip.** Drop `My Clippings.txt` on the Mac
   app, parse the Kindle desktop app's store, and make Kobo a first-class
   two-way target. The honest "two-way" story while Amazon keeps annotations
   locked.
4. **Scribe-aware layout.** Margin room for Active Canvas; ingest the emailed
   ink bundle. Small, growing, high willingness to pay.
5. **Transcript editions.** YouTube and podcast transcripts laid out for
   e-ink with chapter TOC.
6. **Kindle-browser triage page.** Star, skip, or queue items into tonight's
   edition from the device itself. Low effort once the backend exists; keep
   as an experiment, not a pillar.

### 5.5 Risks specific to this bet

- Amazon tunes Send to Kindle policy without notice (the April 2026 address
  change is the latest example). Mitigation: stay on the sanctioned email
  path, never depend on reverse-engineered clients, and keep EPUB export and
  the Kindle app share sheet as fallbacks.
- Instapaper's stated reason for paywalling (per-user parse, image fetch,
  EPUB build, email) is a real cost for a service. It is zero for us, because
  the work runs on the user's device and the mail goes through their account.
  That is the structural reason this can stay free where competitors could
  not.
- "Newspaper" is a metaphor users understand but Kindle's flat TOC limits how
  literal it can be. Prototype the navigation on a real device early.

## 6. UX thesis: where a reader can still shine

Feed lists are commoditised, so the product has to win on how it *feels* to
use, and on making the two-surface model (triage on glass, read on paper)
coherent. Principles to carry into the design doc:

1. **Two surfaces, two jobs.** The phone and Mac are the desk where you decide.
   The e-reader is the room where you read. The app should make that split
   explicit in its structure, not bury it in a settings toggle. Long-form
   should default to "tonight's edition", and reading it on the phone should
   be the exception you opt into.
2. **Editions have a shape.** A front page, sections, a last page. A finite
   object with a name and a date, on every surface. This is what "finite"
   competitors gesture at and only Calibre delivers, and it is the mental
   model that makes "you're done" believable.
3. **Every action answers "where does this go?"** Triage offers exactly three
   verbs: read now, tonight's edition, never. Everything else (star, tag,
   share) is secondary and hidden until wanted.
4. **Seen, not counted.** No badges, no unread numbers, no bold. Items you
   have passed dim and stay in place. A timeline position syncs. This is the
   lesson from the new Reeder's reception.
5. **Delivery you can see.** An edition has a status timeline: built, sent,
   accepted, on device. Failure modes explain themselves in plain language
   with the fix inline. Nobody does this, and it converts the flakiest part of
   the category into a trust signal.
6. **Speed as a feature.** Keyboard-complete on Mac, one-handed swipe on
   iPhone, sub-100ms interactions. The indie Apple clients win on this and we
   should not concede it.
7. **Typography everywhere.** The same care lire and Unread put into on-screen
   type, applied to the EPUB. Per-source profiles that make a blog, a newspaper
   and a docs page each look right on e-ink.
8. **Quiet defaults, no upsell.** The power-user services lose users on AI
   nagging and paywalled basics. Whatever the pricing model, the free or base
   tier must feel complete, and Kindle delivery must not be the paywall
   (Instapaper just showed what happens).

## 7. Positioning in one line

The reader that turns everything you follow into a calm daily edition and
delivers it, reliably and beautifully, to the device you actually read on.

Nearest competitor: Readwise Reader (feeds + read-later + digest, ~$120/yr).
We differ on edition structure, e-ink quality, delivery visibility,
round-trip where possible, Apple-native speed and typography, and on being
free, open source, and account-free with nothing running on our side.

## 8. Decisions this research adds to SCOPE.md §9

- **Licence**: MIT recommended (see §4).
- **Kobo**: file export in Phase 3; the Instapaper API route is out under
  the no-third-party-accounts principle. Highlight import from a mounted Kobo
  on the Mac is cheap and worth doing.
- **Kindle-browser companion**: cut. Dead category, and it would need a
  server to render for the device.
- **Contribution posture toward NetNewsWire**: upstream fixes to shared
  packages where we find them.

## 9. Sources

Readers and features: [NetNewsWire 7.0.4](https://netnewswire.blog/2026/04/03/netnewswire-for-mac-new-icloud.html) ·
[NNW filtering request](https://github.com/Ranchero-Software/NetNewsWire/issues/4406) ·
[Unread](https://www.goldenhillsoftware.com/unread/) ·
[lire review](https://www.macstories.net/reviews/lire-brings-its-highly-customizable-rss-reading-style-to-an-all-new-ipad-design-and-widgets/) ·
[Reeder (new)](https://9to5mac.com/2024/09/05/reeder-launches-new-app-that-unites-rss-with-video-audio-and-social-feeds/) ·
[Reeder reception](https://spectrecollie.com/2025/03/07/i-was-wrong-about-the-new-reeder/) ·
[Feedbin pricing](https://feedbin.com/pricing) ·
[Readwise Reader](https://readwise.io/read) ·
[Inoreader Intelligence](https://www.inoreader.com/blog/2025/04/new-intelligence-reports-and-team-intelligence-plan.html) ·
[Folo](https://github.com/RSSNext/Folo) ·
[Tapestry connectors](https://blog.iconfactory.com/2025/03/supercharge-tapestry-with-connectors/) ·
[feeeed](https://apps.apple.com/us/app/feeeed-rss-reader-and-more/id1600187490) ·
[Current](https://techcrunch.com/2026/02/19/current-is-a-new-rss-reader-thats-more-like-a-river-than-an-inbox/) ·
[Today](https://apps.apple.com/us/app/today-rss-reader/id6754362337) ·
[JorvikDailyNews](https://github.com/PerpetualBeta/JorvikDailyNews) ·
[Instapaper price hike](https://medium.com/@kevinko_60193/instapaper-hikes-premium-prices-f82c71d73445) ·
[Instapaper Kindle paywall](https://www.androidauthority.com/instapaper-send-to-kindle-paywall-imminent-3635014/) ·
[HN on backlogs](https://news.ycombinator.com/item?id=43799697)

Open source: [NetNewsWire](https://github.com/Ranchero-Software/NetNewsWire) ·
[Omnivore](https://github.com/omnivore-app/omnivore) ·
[FeedFlow](https://github.com/prof18/feed-flow) ·
[Readeck iOS](https://github.com/ilyas-hallak/readeck-ios) ·
[FeedKit](https://github.com/nmdias/FeedKit) ·
[SwiftSoup](https://github.com/scinfu/SwiftSoup) ·
[swift-readability](https://github.com/lake-of-fire/swift-readability) ·
[Miniflux](https://github.com/miniflux/v2) ·
[Miniflux Google Reader API](https://miniflux.app/docs/google_reader.html) ·
[FreshRSS](https://github.com/FreshRSS/FreshRSS) ·
[Feedbin](https://github.com/feedbin/feedbin) ·
[Feedbin Kindle](https://feedbin.com/help/sharing-read-it-later-services/) ·
[Readeck](https://github.com/readeck/readeck) ·
[Wallabag](https://github.com/wallabag/wallabag) ·
[Shiori](https://github.com/go-shiori/shiori) ·
[Folo self-host issue](https://github.com/RSSNext/Folo/issues/4164) ·
[RSSHub](https://github.com/DIYgod/RSSHub) ·
[RSS-Bridge](https://github.com/RSS-Bridge/rss-bridge) ·
[Kill the Newsletter](https://github.com/leafac/kill-the-newsletter) ·
[Defuddle](https://github.com/kepano/defuddle) ·
[trafilatura](https://github.com/adbar/trafilatura) ·
[go-trafilatura](https://github.com/markusmobius/go-trafilatura) ·
[extraction benchmark](https://github.com/scrapinghub/article-extraction-benchmark) ·
[go-epub](https://github.com/go-shiori/go-epub) ·
[KindleEar](https://github.com/cdhigh/KindleEar) ·
[RSSPub](https://github.com/harshit181/RSSPub) ·
[inkfeed](https://github.com/adhamsalama/inkfeed) ·
[Calibre ebook-convert](https://manual.calibre-ebook.com/generated/en/ebook-convert.html)

Kindle: [Readwise exporting](https://docs.readwise.io/reader/docs/faqs/exporting) ·
[Readwise Kindle import](https://docs.readwise.io/readwise/docs/importing-highlights/kindle) ·
[Send to Kindle formats](https://www.amazon.com/gp/help/customer/display.html?nodeId=G7NECT4B4ZWHQ8WV) ·
[Send to Kindle limits](https://www.amazon.com/gp/help/customer/display.html?nodeId=G5WYD9SAF7PGXRNA) ·
[Full-address requirement, April 2026](https://tech.yahoo.com/general/articles/send-kindle-soon-require-full-130420868.html) ·
[KDP nav guidelines](https://kdp.amazon.com/en_US/help/topic/GY3AD8C6C6GAG42N) ·
[Personal-doc highlights export](https://mjtsai.com/blog/2024/11/05/exporting-kindle-highlights-for-personal-documents/) ·
[Kindle browser 5.16.4](https://goodereader.com/blog/electronic-readers/amazon-has-a-better-internet-browser-on-kindle-e-readers) ·
[Reabble](https://reabble.com/en/) ·
[Scribe Active Canvas](https://goodereader.com/blog/kindle/kindle-scribe-1-to-get-active-canvas-and-extended-margins-in-2025) ·
[Kobo Instapaper firmware](https://blog.the-ebook-reader.com/2025/08/26/latest-kobo-software-update-adds-instapaper-support/) ·
[FiveFilters Push to Kindle](https://fivefilters.gumroad.com/l/push-to-kindle) ·
[Send to Kindle extension reviews](https://chromewebstore.google.com/detail/send-to-kindle-for-google/cgdjpilhipecahhcilnafpblkieebhea) ·
[stkclient](https://github.com/maxdjohnson/stkclient)
