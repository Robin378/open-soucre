# SportsHub — Flutter Android App Build Specification

**You are an expert Flutter/Android developer. Build the app described below exactly as specified.** This document is the single source of truth — data models, API contracts, screen specs, and design tokens are all final. Where something is genuinely ambiguous, pick the most standard Flutter pattern and note the assumption in a code comment rather than stopping.

---

## 0. How to Use This Document

1. Paste this entire file into the agent (Gemini / Google AI Studio "Antigravity") as the build prompt.
2. Also attach the four original HTML mockups (`AppEventPage.html`, `AppCetagoryPage.html`, `AppSportsPage.html`, `AppDrawerPage.html`) — they are the pixel/style reference. This document tells you *what to build*; the HTML files show *exactly what it should look like*.
3. Build in the order given in **Section 13**.

---

## 1. App Overview

| | |
|---|---|
| **App name** | SportsHub |
| **Brand / publisher** | RORAX |
| **Suggested package id** | `com.rorax.sportshub` |
| **Platform** | Android, Flutter (single codebase, Android target only for now) |
| **Purpose** | A dark-themed sports schedule + IPTV channel-browsing app. Three tabs: **Events** (SofaScore fixture schedule), **Categories** (IPTV channels grouped by category), **Sports** (dedicated sports-channel grid). |
| **No login / no accounts** | Fully anonymous, no backend auth. |

The app has two completely independent data sources feeding two completely independent features — do not conflate them:

- **Events tab** → live data pulled at runtime from the **SofaScore public API** (fixtures/schedule only, no scores).
- **Categories tab + Sports tab** → IPTV data pulled from **Robin's `database.json`** on GitHub, which lists *playlist packs* (not individual channels) — each pack points to a separate `.m3u8` file that the app fetches and parses on demand. See Section 5.1 for the exact pipeline.

---

## 2. Tech Stack

- **Flutter** (stable channel), Android only
- **State management:** Riverpod (`flutter_riverpod`)
- **Networking:** `dio`
- **Video playback (HLS/M3U8):** `better_player_plus` (actively maintained fork of better_player; handles HLS, quality/track selection, and PiP better than plain `video_player`)
- **Fonts:** `google_fonts` → Rajdhani (weights 500/600/700)
- **Icons:** `lucide_icons` for the generic line-icon set (closest open-source match to the hand-drawn stroke icons in the mockups) + `flutter_svg` for the RORAX logo and any icon that must be pixel-exact
- **Images/logos:** `cached_network_image`
- **Local storage:** `shared_preferences` (settings, favorites, custom playlist URLs)
- **Utilities:** `intl` (date/time formatting), `url_launcher` (Telegram/Website/Contact links), `share_plus` (Share App)

---

## 3. Design System

The three page mockups use *slightly* inconsistent token values (e.g. `--bg-main` is `#0d0d10` on Categories/Sports but `#0a0a0e` on Events). **Standardize on the Events page's richer token set across the whole app** so every screen looks consistent:

```dart
class AppColors {
  static const bgMain      = Color(0xFF0A0A0E);
  static const bgCard      = Color(0xFF16171F);
  static const bgCardTop   = Color(0xFF1B1C26); // gradient top-left of cards
  static const borderAccent= Color(0xFF4A5DFB); // primary blue accent
  static const borderSoft  = Color(0x474A5DFB); // border-accent @ 28% opacity
  static const badgeBg     = Color(0xFF0D0D12);
  static const textPrimary = Color(0xFFF5F5F7);
  static const textSecondary = Color(0xFF9A9AA5);
  static const textDate    = Color(0xFF6C9BFF);
  static const accentPink  = Color(0xFFFF5D8F); // live indicator only
  static const ribbonBg    = Color(0xFFFF3B3B); // "HOT"/"TOP" ribbons
  static const circleBg    = Color(0xFF2A2B38);
}
```

- **Font:** Rajdhani, weight 600 as body default, 700 for titles/labels, 500 for the lightest secondary text.
- **Corner radius:** 16px on all cards (`--radius-card`).
- **Card border:** 1px `borderSoft` on list cards (Events match cards), 1–2px solid `borderAccent` on grid cards (Category/Sports tiles).
- **Background wash:** subtle radial blue glow at top-center + faint pink glow top-right, over `bgMain` (see Events mockup `body` background) — apply this once at the app-shell level so it's consistent across tabs.
- **Icons:** thin-stroke (1.5–2px), no fill, rounded joins/caps — matches Lucide's default style closely.
- **Bottom nav:** always 3 items — Events / Categories / Sports — active item + label tinted `borderAccent`, inactive tinted `textSecondary`.
- **Header pattern (all 3 tabs):** hamburger icon (opens Drawer) — title — [star, refresh, search] icon cluster on the right. On the Events tab the title is the app name "SportsHub"; on the other two tabs the title is the section name ("Categories", "Sports").

---

## 4. Information Architecture

```
App Shell (bottom nav, persists across tabs)
├── Events           (SofaScore schedule)
├── Categories        (IPTV categories grid)
│    └── Channel List (tap a category → list of channels in it)
│         └── Player  (tap a channel → full-screen HLS playback)
└── Sports            (IPTV sports-channel grid, direct → Player)

Drawer (hamburger icon, any tab)
├── Network Stream     → play an ad-hoc M3U8 URL the user pastes in
├── Playlists          → manage saved custom M3U/M3U8 playlist URLs
├── Floating Player     → toggle: playback continues in a small draggable overlay (Android PiP)
├── Force Low Quality   → toggle: caps HLS playback to lowest available bitrate
├── Cricket Score       → external link (configurable URL) for full live scores
├── Football Score      → external link (configurable URL) for full live scores
├── Telegram            → opens Telegram (default: @my_dad_raaj — edit in constants.dart)
├── Contact US          → mailto: or Telegram, per constants.dart
├── Website             → opens rorax.xyz in an in-app WebView
├── Share App           → native share sheet with Play Store link
├── Copyright           → simple about/copyright dialog
└── Exit                → confirmation dialog → SystemNavigator.pop()
```

Header icon cluster (Events/Categories/Sports, top-right): **star = Favorites filter**, **circular-arrow = manual refresh**, **magnifying glass = in-list search** (client-side filter of whatever's already loaded — no extra API calls).

---

## 5. Data Sources

### 5.1 IPTV Channel Database (database.json + per-pack M3U8 files)

**Live URL:** `https://raw.githubusercontent.com/robinhossainraaj/rorax-iptv-database/refs/heads/main/database.json`

This is **not** a flat list of channels — it's a list of **playlist packs**. Each pack is a category-tagged bundle that points to a separate `.m3u8` file; the actual channels only exist inside that `.m3u8` file and have to be fetched + parsed separately. Current real shape:

```json
[
  {
    "name": "FIFA World Cup 2026",
    "category": "Sports",
    "description": "Official broadcasters and sports channels with FIFA World Cup 2026...",
    "playlist": "https://raw.githubusercontent.com/.../FIFA_WorldCup2026_Channels_fixed.m3u8",
    "channels": 50,
    "updated": "2026-05-24"
  },
  {
    "name": "Global Sports Hub 2026",
    "category": "Sports",
    "description": "Stream 160+ live channels from Europe, Latin America, Middle East...",
    "playlist": "https://raw.githubusercontent.com/.../Global_Sports_Hub_2026_fixed.m3u8",
    "channels": 160,
    "updated": "2026-05-31"
  }
]
```

**Field notes:**
- `category` is the string that drives the **Categories** grid — derive the distinct category list from all packs at runtime (don't hardcode it). Map known category names to an icon locally (e.g. `"Sports"` → circle-dot, `"Multi - Channels"` → layers, unmapped → generic folder icon).
- `playlist` is a URL to a raw `.m3u8` file — fetch its **text content** and parse it client-side (Section on M3U parsing below). This is not JSON.
- `channels` is a manually-typed count in the source file — treat it as a hint only, never as ground truth for a UI badge. Always show the *actual* parsed channel count once a pack's playlist has been fetched.
- `updated` is a plain date string, display-only.
- **Sports tab = Categories tab pre-filtered to `category == "Sports"`.** It is not a separate flag on individual channels — it's literally "every pack tagged Sports, flattened into one grid," reusing the exact same fetch/parse path as tapping "Sports" from the Categories grid. Don't build two separate data paths for this.

**⚠️ Two data-quality issues in the live file right now — flag both to Robin, the app should defend against the first and the source file needs fixing for the second:**
1. **The JSON is currently invalid.** The array entries are missing commas between the closing `}` of one pack and the opening `{` of the next (e.g. `...}\n{...` instead of `...},\n{...`). A plain `jsonDecode` will throw on this file as it stands today. Two things to do: (a) have the repository's publishing step actually emit valid JSON going forward — this is the real fix; (b) as a defensive fallback in the app, wrap the parse in try/catch and, on failure, attempt a lenient repair (regex-replace `}\s*\n\s*{` → `},\n{` before re-parsing) so the app degrades gracefully instead of hard-crashing on a bad publish. Surface a visible "database has a formatting issue" error state if even the repaired parse fails — don't fail silently.
2. **Mojibake in `description` text** — e.g. `âš½ðŸ”¥` where the intent was clearly emoji (⚽🔥). This is a UTF-8 double-encoding issue from however the file was last saved/edited. Cosmetic, not a parser blocker, but worth fixing at the source so descriptions don't render garbled.
3. Also worth a manual look: two consecutive packs are both named **"Combined BD/IND International Channels"** with identical `category`/`description`/`channels` count but two different `playlist` URLs (one third-party repo, one Robin's own). If that's intentional (two independent sources merged under one display name), fine — but if it's a leftover duplicate from editing, it'll show as two identical-looking cards in whatever list surfaces individual packs, so worth confirming.

**M3U8 parsing (the actual channel extraction):**

Each pack's `.m3u8` is standard Extended M3U:
```
#EXTM3U
#EXTINF:-1 tvg-id="espn.us" tvg-logo="https://.../espn.png" group-title="Sports",ESPN
https://.../espn-stream.m3u8
#EXTINF:-1 tvg-logo="https://.../beinsports.png" group-title="Sports",beIN Sports 1
https://.../bein1-stream.m3u8
```

Parsing algorithm (write a small custom parser — don't pull in an unmaintained pub package for something this simple):
1. Split the fetched text into lines, normalize CRLF/LF.
2. Iterate lines; when a line starts with `#EXTINF`, extract `tvg-logo="([^"]*)"` and `group-title="([^"]*)"` via regex, and take everything after the **last comma** on that line as the display name.
3. Between the `#EXTINF` line and the URL line there may be one or more `#EXTVLCOPT:http-user-agent=...` / `#EXTVLCOPT:http-referrer=...` lines — capture these as the channel's `userAgent`/`referer` overrides if present (many IPTV sources need a spoofed Referer/User-Agent to play at all — this is the same class of issue as SofaScore requiring a browser User-Agent, just per-stream instead of per-API). If absent, leave both null and let the player fall back to app-wide defaults.
4. The next non-empty, non-`#`-prefixed line is the stream URL for that entry.
5. Skip `#EXTM3U` and any other `#EXT-X-*` lines.
6. If `tvg-logo` is missing, fall back to a generic channel placeholder icon (don't leave it blank).
7. A per-channel `group-title` (from inside the M3U8 itself) can differ from the pack's own `category` field — surface it as a small secondary tag on the channel row if present, but don't use it to create extra top-level navigation; the pack's `category` is what drives Categories/Sports tab placement.

**Fetch/cache strategy:**
- Fetch `database.json` on app start (with the lenient-repair fallback above) → cache parsed packs to `shared_preferences` with a timestamp → show cached data instantly next launch while refetching in the background.
- Do **not** eagerly fetch every pack's `.m3u8` on startup — that could mean parsing 269+ line playlists across multiple packs before the user has picked anything. Fetch + parse a pack's `.m3u8` only when its category is opened (Categories → tap category, or the Sports tab is first opened), then cache the parsed channel list in memory for the session so revisiting the same category doesn't re-fetch/re-parse.
- Pull-to-refresh and the header refresh icon re-fetch `database.json`; if channels for the currently open category are already cached, also refetch just that category's underlying `.m3u8` files.

### 5.2 SofaScore Sports Schedule API

Unofficial, undocumented, public JSON API. No key required, but Cloudflare will reject requests without a browser-like `User-Agent`.

```
Base URL:  https://api.sofascore.com/api/v1
Endpoint:  GET /sport/{sport}/scheduled-events/{yyyy-MM-dd}
Sports:    football | cricket
Header:    User-Agent: Mozilla/5.0   ← required, requests without it can 403
```

Example: `https://api.sofascore.com/api/v1/sport/football/scheduled-events/2026-07-15`

This returns **one calendar day at a time** — the Events screen needs a date selector (Today / Tomorrow / a short forward-looking date strip) that re-queries per selected date, per sport.

**Response shape (trimmed to the fields you need):**
```json
{
  "events": [
    {
      "id": 10461276,
      "startTimestamp": 1658246400,
      "status": { "type": "notstarted" },
      "tournament": {
        "name": "FIFA World Cup, Group Stage",
        "category": { "name": "World" }
      },
      "homeTeam": { "name": "Brazil", "national": true },
      "awayTeam": { "name": "Argentina", "national": true }
    }
  ]
}
```

- `startTimestamp` is **Unix seconds** → `DateTime.fromMillisecondsSinceEpoch(startTimestamp * 1000)`.
- `status.type` is one of `notstarted` / `inprogress` / `finished`. **Do not display scores or a score field at all** — the app only ever shows kickoff time, teams, and competition. It's fine (and useful) to show a small "LIVE" text tag when `status.type == "inprogress"`, with no score attached, but this is optional polish, not a requirement.
- **International-only filter:** keep an event only if **both** `homeTeam.national == true` and `awayTeam.national == true`. This is SofaScore's own flag for "this is a national team, not a club side" and is the correct way to isolate international fixtures (World Cup, Euro/Copa América/Asian Cup, Nations League, T20/ODI World Cup, bilateral international series, etc.) from club/league matches.
  - ⚠️ **Verify this for cricket specifically** with a live test call before shipping — the `national` flag is confirmed present and reliable for football; cricket team objects should be spot-checked the same way. If it's missing/unreliable for cricket, fall back to filtering on `tournament.category.name` against a small allow-list (e.g. `"International"`, `"World"`, `"ICC Cricket World Cup"`, `"T20 World Cup"`, `"Champions Trophy"`, `"Asia Cup"`).
- Sort the merged football+cricket list by `startTimestamp` ascending.

**Caching:** cache each `(sport, date)` response in memory for the session — no need to persist across app restarts, schedules change daily anyway.

---

## 6. Data Models

```dart
/// One entry from database.json — a bundle of channels, not a channel itself.
class IptvPack {
  final String name;
  final String category;       // drives Categories grid + Sports-tab filter
  final String description;
  final String playlistUrl;    // .m3u8 to fetch + parse on demand
  final int declaredChannelCount; // from source JSON — display as a hint only
  final String updated;
  IptvPack({
    required this.name,
    required this.category,
    required this.description,
    required this.playlistUrl,
    required this.declaredChannelCount,
    required this.updated,
  });
}

/// Derived, not stored — computed from the distinct `category` values across packs.
class Category {
  final String name;
  final String icon; // local lookup by name, e.g. "Sports" -> circle-dot
  final int packCount;
  Category({required this.name, required this.icon, required this.packCount});
}

/// One entry parsed out of a pack's .m3u8 file.
class Channel {
  final String name;
  final String? logo;          // tvg-logo, nullable — use placeholder if missing
  final String streamUrl;
  final String? groupTitle;    // per-channel group-title from the M3U8 itself, optional
  final String? userAgent;     // from #EXTVLCOPT:http-user-agent=, if present
  final String? referer;       // from #EXTVLCOPT:http-referrer=, if present
  final String sourceCategory; // inherited from the parent pack's `category`
  final String sourcePackName; // which pack this came from, useful for de-duping/debug
  Channel({
    required this.name,
    this.logo,
    required this.streamUrl,
    this.groupTitle,
    this.userAgent,
    this.referer,
    required this.sourceCategory,
    required this.sourcePackName,
  });
}

enum SportType { football, cricket }
enum MatchStatus { notStarted, inProgress, finished }

class Match {
  final int id;
  final SportType sport;
  final String tournamentName;
  final String competitionCategory; // tournament.category.name
  final String homeTeam;
  final String awayTeam;
  final DateTime kickoff; // from startTimestamp
  final MatchStatus status;
  Match({
    required this.id,
    required this.sport,
    required this.tournamentName,
    required this.competitionCategory,
    required this.homeTeam,
    required this.awayTeam,
    required this.kickoff,
    required this.status,
  });
}
```

---

## 7. State Management (Riverpod)

```dart
// Data providers
final packsProvider = FutureProvider<List<IptvPack>>((ref) async {
  // fetch database.json, jsonDecode with lenient-repair fallback (Section 5.1)
});

final categoriesProvider = Provider<List<Category>>((ref) {
  final packs = ref.watch(packsProvider).value ?? [];
  // group `packs` by `category`, count packs per category, attach icon lookup
});

// Fetches + parses every pack's .m3u8 for a given category, flattens to one channel list.
// Reused as-is by both "Categories -> tap a category" and the Sports tab
// (Sports tab = this provider called with categoryName: "Sports").
final channelsForCategoryProvider =
    FutureProvider.family<List<Channel>, String>((ref, categoryName) async {
  final packs = (ref.watch(packsProvider).value ?? [])
      .where((p) => p.category == categoryName);
  // fetch + parse each pack.playlistUrl in parallel, flatten, cache in memory
});

final sportsChannelsProvider = FutureProvider<List<Channel>>((ref) {
  return ref.watch(channelsForCategoryProvider('Sports').future);
});

// Events tab
final selectedDateProvider = StateProvider<DateTime>((ref) => DateTime.now());
final selectedSportProvider = StateProvider<SportType?>((ref) => null); // null = both
final matchesProvider = FutureProvider.autoDispose<List<Match>>((ref) async {
  final date = ref.watch(selectedDateProvider);
  final sport = ref.watch(selectedSportProvider);
  // fetch football and/or cricket for `date`, merge, filter international-only, sort
});

// Cross-cutting
final searchQueryProvider = StateProvider<String>((ref) => '');
final favoritesProvider = StateNotifierProvider<FavoritesNotifier, Set<String>>(...);
final forceLowQualityProvider = StateProvider<bool>((ref) => false); // shared_preferences-backed
```

---

## 8. Screens

### 8.1 App Shell
`Scaffold` with `drawer:` set to the Drawer widget, `bottomNavigationBar:` with the 3 tabs, `IndexedStack` body so tab state (scroll position, filters) survives switching tabs.

### 8.2 Events Screen
Reference: `AppEventPage.html`. Key differences from the mockup to implement:
- Category-icon row → only **2** icons: Football, Cricket (not 4). Tapping toggles `selectedSportProvider` (tap again / tap the other → both selected = `null`).
- Replace the `Live / Upcoming / Finished` filter pills with a **horizontal date strip** (Today, Tomorrow, +2…+6 days) — this drives which date is queried from SofaScore, since the API is per-day.
- Match card: **no score, no live score row.** Center column shows kickoff time (local device timezone, formatted e.g. "8:30 PM"), date, and a small competition/tournament badge above (as in the mockup). If `status == inProgress`, optionally show a small "LIVE" pink tag with no numbers — no live pulsing score UI.
- Empty state: "No international {sport} matches on {date}" with the announcement-banner style container reused as an empty-state card.
- Pull-to-refresh + header refresh icon both re-trigger `matchesProvider`.

### 8.3 Categories Screen
Reference: `AppCetagoryPage.html` — exact 3-column grid, circular icon + label card, as shown. Data comes from `categoriesProvider` (derived from `database.json` pack categories, not a separate categories file); each card's icon maps from `Category.name` via a local name→icon lookup table (fallback to a generic folder/tag icon if unmapped). Card label = `Category.name`. Tapping a card pushes **Channel List**, passing the category name. Show a loading skeleton while `packsProvider` resolves.

### 8.4 Channel List Screen (new — not in the mockups, derived)
Not covered by an existing mockup — build it in the same visual language as the other screens: dark background, `bgCard` rows with `borderSoft`, `radius-card` 16px. Simple vertical list: circular channel logo (44px, `circleBg` fallback if no logo) — channel name (+ small `groupTitle` tag if present) — chevron. AppBar title = category name, back button, same header icon cluster minus star (search still useful here). Data comes from `channelsForCategoryProvider(categoryName)` — this triggers fetching + parsing every pack's `.m3u8` under that category, so show a loading state (this can take a moment for larger packs like the 269-channel one) and a partial-failure state if one pack's playlist fails to fetch/parse while others succeed (show the ones that worked, don't blank the whole screen). Tapping a row pushes **Player**, passing the `Channel`.

### 8.5 Sports Screen
Reference: `AppSportsPage.html` — same 3-column grid component as Categories, but each card is a **channel**, sourced from `sportsChannelsProvider` (i.e. `channelsForCategoryProvider('Sports')` — the exact same fetch/parse path as tapping "Sports" on the Categories tab, just displayed as a permanent flat grid instead of a pushed list). Card label = `Channel.name`, logo = `Channel.logo`. Tapping a card pushes **Player** directly (no intermediate list, since this tab *is* the flat list). Same loading/partial-failure handling as 8.4.

### 8.6 Video Player Screen (new — derived from drawer features)
Full-screen `better_player_plus` HLS player.
- Source: `Channel.streamUrl`, with `Channel.userAgent`/`referer` passed as custom headers if present, else app defaults.
- If `forceLowQualityProvider` is on, select the lowest-bitrate HLS variant/track if the manifest exposes multiple.
- "Floating Player" toggle (drawer or in-player button) → shrinks playback into a small draggable overlay using Android native Picture-in-Picture (`better_player_plus` has PiP support) so playback continues while browsing other tabs.
- Standard controls: play/pause, seek bar (live streams can hide/disable seek), fullscreen toggle, retry-on-error.

### 8.7 Drawer & Sub-Screens
Reference: `AppDrawerPage.html` — reproduce exactly: black background, RORAX triangle logo (top-left, use the literal SVG path from the mockup via `flutter_svg` for pixel accuracy), "Version: 2.2" label, then the 12-item list with icon + label rows exactly as ordered in the mockup. Wire each item per the Information Architecture table in Section 4. Keep icon style consistent (white stroke, 22×22, 1.8 stroke width) matching the mockup's inline SVGs — use `flutter_svg` with the mockup's raw path data for a 1:1 match rather than approximating with a generic icon pack here, since these are custom-drawn.

---

## 9. Networking, Caching, Error & Empty States

- Wrap every API/JSON call in try/catch; surface failures as a retry-able error card (icon + message + "Retry" button), never a blank screen or raw exception text.
- Timeouts: 10s connect / 15s receive on `dio` for SofaScore, the `database.json` fetch, and each pack's `.m3u8` fetch.
- If SofaScore is unreachable, keep the Categories/Sports tabs fully functional — the two data sources are independent and one failing must never block the other.
- `database.json` parse failure (including the known missing-comma issue, Section 5.1): try the lenient repair once; if that also fails, show a clear "channel database couldn't be read" error state with Retry — don't crash the app.
- A single pack's `.m3u8` failing to fetch/parse should not take down the whole category: skip that pack, show the channels from packs that did succeed, and optionally show a small inline notice that one source is temporarily unavailable.
- Loading state: skeleton/shimmer placeholders matching card shapes, not a bare spinner, for Events/Categories/Sports grids. The Channel List / Sports grid loading state should account for `.m3u8` parsing taking noticeably longer than a small JSON fetch on the larger packs (160–270 channels).

---

## 10. Android Configuration

- `AndroidManifest.xml`: `INTERNET` permission.
- If any M3U8 sources are plain `http://` (not `https://`), add a `network_security_config.xml` allowing cleartext traffic for those specific domains (don't blanket-allow cleartext for all domains).
- `better_player_plus` / ExoPlayer handles HLS natively — no extra manifest entries needed beyond the plugin's own setup steps.
- Minimum SDK: whatever `better_player_plus` currently requires (check its pub.dev page at build time — don't hardcode a number here, plugin minimums change).

---

## 11. Package List (pubspec.yaml — names only, agent should resolve latest compatible versions)

```yaml
dependencies:
  flutter_riverpod:
  dio:
  better_player_plus:
  google_fonts:
  lucide_icons:
  flutter_svg:
  cached_network_image:
  shared_preferences:
  intl:
  url_launcher:
  share_plus:
  webview_flutter:
```

---

## 12. Folder Structure

```
lib/
├── main.dart
├── core/
│   ├── constants.dart        // API base URLs, GitHub JSON URLs, drawer link targets
│   ├── theme.dart             // AppColors, text styles, Rajdhani setup
│   └── network/dio_client.dart
├── models/
│   ├── iptv_pack.dart
│   ├── category.dart
│   ├── channel.dart
│   └── match.dart
├── data/
│   ├── iptv_repository.dart   // database.json fetch (+ lenient repair) + cache, category derivation
│   └── sofascore_repository.dart // scheduled-events fetch + filter + mapping
├── utils/
│   └── m3u_parser.dart        // parses a pack's .m3u8 text into List<Channel>, per Section 5.1
├── providers/
│   └── providers.dart         // all Riverpod providers from Section 7
├── screens/
│   ├── shell/app_shell.dart
│   ├── events/events_screen.dart
│   ├── categories/categories_screen.dart
│   ├── categories/channel_list_screen.dart
│   ├── sports/sports_screen.dart
│   ├── player/player_screen.dart
│   └── drawer/
│       ├── app_drawer.dart
│       ├── network_stream_screen.dart
│       └── playlists_screen.dart
└── widgets/
    ├── match_card.dart
    ├── category_card.dart
    ├── channel_tile.dart
    └── empty_state.dart
```

---

## 13. Build Order for the Agent

1. Project scaffold + theme (`AppColors`, Rajdhani via `google_fonts`) + app shell with 3-tab bottom nav and `IndexedStack`.
2. Data models (Section 6) + `iptv_repository.dart` + `sofascore_repository.dart` (Section 5), with the SofaScore fetch, unit-testable against the sample JSON in this doc.
3. Riverpod providers (Section 7).
4. Categories screen (simplest — one grid, one data source) → Channel List screen → Sports screen (reuses the same grid card widget as Categories).
5. Player screen wired from both Categories→ChannelList and Sports.
6. Events screen (most complex: date strip, sport toggle, international filter, empty/loading states).
7. Drawer + its sub-screens, wired per Section 4.
8. Error/empty/loading polish pass (Section 9) across all screens.
9. Manual test pass on a real device/emulator: verify HLS playback, PiP floating player, and that a SofaScore outage doesn't break the Categories/Sports tabs.

---

## 14. Phase 2 / Stretch Backlog

Not required for MVP — wire the drawer entries so tapping them doesn't crash (e.g. simple "coming soon" screen), but build fully only after the core 3 tabs + player are solid:

- Favorites (star icon) — persisted list of favorited categories/channels/matches, filterable view.
- Network Stream / Playlists — client-side M3U/M3U8 parsing so a user can paste any playlist URL and have it generate ad-hoc categories/channels, without touching the GitHub-hosted database.
- Force Low Quality — real HLS variant/track selection rather than just capping resolution.
- AdMob monetization (banner/interstitial/rewarded) — not scoped in this document; revisit separately once the core app is stable.

---

## 15. Open Questions / Things to Verify Before Shipping

- Confirm `national` flag reliability for **cricket** team objects specifically (confirmed reliable for football).
- Decide final destination URLs for "Cricket Score" / "Football Score" / "Telegram" / "Contact US" / "Website" drawer items — placeholders are in `constants.dart`, fill in before release.
- **Fix `database.json` at the source** so it's valid JSON (missing commas between array entries, Section 5.1) — the app-side lenient repair is a safety net, not a substitute for publishing valid JSON.
- Fix the mojibake in pack `description` text (re-save as proper UTF-8).
- Confirm whether the two "Combined BD/IND International Channels" entries are intentionally separate sources or a leftover duplicate.
- As more categories get added to `database.json` beyond `"Sports"` / `"Multi - Channels"`, extend the name→icon lookup table used by the Categories screen so new categories don't silently fall back to the generic icon.
