# Juxtopposed redesign: implementation and API map

Status: planning and source audit, not an implemented redesign. Audited 7 October 2026 against Spotifast `995c768dcba4da7f93302bf2107af6b7ec3df02b` (0.12.0).

Target: [Aru's Spotifast fork](https://github.com/arugyani/spotifast). Design: [Juxtopposed's Spotify Redesign, supplied Figma copy](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/Spotify-Redesign--Community-), Redesign page `3:2`. Original: [community file](https://www.figma.com/community/file/1376999463181735262/spotify-redesign).

## Decision

The redesign is feasible as a native Rust/egui implementation. There is no JavaScript, CSS, or arbitrary layout plugin interface in this repository. Custom JSON themes change colors; Winamp `.wsz` skins apply to the separate mini player. Neither changes the main application's layout, navigation, or feature set.

The happy path is to reuse the existing playback, action, API, caching, and account machinery while replacing the native presentation and adding the missing pages and local state. Most music-library interactions already have a backend. The Figma file also depicts Spotify-only services that are not supplied by the current integrations. A fork can add client features and new supported integrations; a fork alone does not provide access to Spotify's private services.

This map treats upstream product exclusions as decisions we can revisit. Local folders, local bookmarks, feed customization, sleep timers, and richer layouts are engineering work. They are not automatically blocked because upstream does not want them. Spotify-synchronized folders, friend activity, Spotify DJ, and video playback have different dependencies and must not be represented as working through dummy data.

## How to read the map

| Mark | Meaning |
| --- | --- |
| **Reuse** | Relevant backend/action already exists. The design still needs native UI work. |
| **Local** | Implementable inside this fork without a new Spotify service. |
| **API gap** | A documented API exists, but its model, request, response, or UI integration is missing here. |
| **Conditional** | Depends on grant, account, market, active device, playlist permissions, or a non-public session capability. |
| **External** | No supported path identified in the audited Web API and pinned player. Needs another provider or demonstrated protocol/player support. |
| **Unspecified** | Figma names a control or destination but does not supply enough behavior/content to define it completely. |

These are engineering classifications, not live-account test results. No authenticated Spotify capability tests were performed for this audit. A documented endpoint's existence does not prove that a particular client ID can call it.

The [machine-readable inventory](docs/redesign/figma-inventory.json) includes every top-level node on the supplied Redesign page, its Figma link, dimensions, and immediate component states. It complements the feature mapping below. It is not an export of all nested prototype reactions or a copy of the sample lyrics/artwork.

## 1. Existing architecture and extension points

| Layer | Existing source | Reuse and extension |
| --- | --- | --- |
| Shell and page composition | [UI root](src/ui/mod.rs), [top bar](src/ui/topbar.rs), [sidebar](src/ui/sidebar.rs), [player bar](src/ui/player_bar.rs) | Recompose header, library rail, main panel, player, and drawers. Keep existing keyboard, account, update, and window behavior reachable. |
| Tokens, fonts, icons | [theme](src/theme.rs), `fastframe-theme`, `fastframe-fonts`, `fastframe-icons` in [Cargo.toml](Cargo.toml) | Add design geometry/typography alongside semantic palette tokens. The current main font is Inter and icons are Lucide; Figma uses Satoshi and its own icon assets. Exact assets/font licensing and bundling are implementation dependencies. |
| Shared UI | [widgets](src/ui/widgets.rs), [collection tables and hero](src/ui/collection.rs), [dialogs](src/ui/dialogs.rs) | Extend cards, shelves, rows, chips, menus and cover presentation. Preserve virtualized lists, keyboard selection, drag/drop and bidirectional text. |
| Routing and actions | `Page`, `Action`, `Library`, `HomeData`, `Loadable`, `PagedList` in [model](src/model.rs) | Add routes and state for Library, Discover, Song, Episode, Profile and audiobook browsing. Persist new route encodings without breaking old ones. |
| State changes | [app](src/app.rs) | Apply UI actions after drawing. Preserve optimistic mutations, request generations, account boundaries, queue order and reconnection behavior. |
| Async work | `ApiRequest`/`ApiResponse` in [backend](src/backend.rs) | Add missing requests off the UI thread. Keep cancellation, paging, caching and stale-response rejection. |
| HTTP and grant selection | [client](src/api/client.rs), [gateway](src/api/gateway.rs), [models](src/api/models.rs), [auth](src/auth.rs) | Extend current models and requests. Use capability-aware routing; do not send every new request through the personal app by default. |
| Session and audio | [session reads](src/session_reads.rs), [player](src/player.rs) | Reuse playlist reads, rootlist, lyrics, radio, Connect and playback. Audio-engine changes belong here, separate from the redesign. |
| Persistent preferences | [settings](src/settings.rs) | Extend `HomeSettings`, account-scoped library organization, selected tabs and views. Defaults and migration must keep existing settings readable. |

Existing routes cover Home, Top Songs, Search, Liked Songs, Albums, Artists, Podcasts, Episodes, individual playlists/albums/artists/shows, Radio, Queue and Settings. A `Track` or `Episode` API request already exists, but a dedicated Song or Episode page does not. This distinction prevents inventing unnecessary new API calls.

## 2. Desktop screen map

All 16 desktop frames are 1771 × 1024. Every screen inherits the shell and player requirements in section 4. Links open the exact Figma node.

| Screen / Figma node | Existing implementation | Required change and support |
| --- | --- | --- |
| [Home](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-16461), [alternate Home](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-16850) | `ui/home.rs`, `HomeData`, `HomeSettings` | **Reuse + Local + Conditional.** Replace greeting/quick-access layout with content chips and ordered shelves, stacked playlist covers, carousel actions and feed customization. Expand the two existing shelf visibility settings. Exact Spotify home recommendations and personalized mixes are not provided by a general Home API. See H1-H6 below. |
| [Search](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-17590) | `ui/search.rs`, `SearchState`, split catalogue/playlist requests | **Reuse + Local + Conditional.** Add artwork-backed recent items and grouped Browse all tiles. Preserve live search, filters, stale-query cancellation and partial results. Curated genre/mood tiles can launch searches; those are not Spotify's private browse feed. |
| [Library grid](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-18046) | `ui/library.rs`, sidebar collections, library models | **Local.** Add unified mixed-type library route, content filter, sort, search and view controls. Reuse the existing collections. Folder and audiobook entries need the additions below. |
| [Library list](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-34928) | Collection table and sidebar rows | **Local.** Share the same filtered collection with grid and compact modes. Use type-appropriate columns rather than fabricated album/duration values for artists or folders. |
| [Song](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-18150) | `ApiRequest::Track`, lyrics panel, item menus, radio | **Local + External.** Add `Page::Song`, metadata cache, header and Lyrics/Credits/More like this tabs. Track, album, duration and primary artists exist. Global play counts, detailed personnel credits and Genius annotations do not. |
| [Album](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-18668) | `ui/collection.rs::album`, `AlbumPage` | **Reuse.** Move hero/control hierarchy and artwork/metadata to the designed positions; reuse tracks, play, save, queue and credits/copyright available from album data. Genre and contributor chips must use real metadata, not the sample labels. |
| [Playlist](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-18763) | `ui/collection.rs::playlist`, `PlaylistPage` | **Reuse + Conditional.** Restyle header, tracks and right-hand information. Keep edit permissions, duplicates, snapshots, multi-selection, reorder and pagination. Editable versus followed/editorial playlists must remain distinct. |
| [Artist Home](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-19023) | `ui/artist.rs`, `ArtistPage` | **Reuse + Local + External.** Add banner, tab strip, personal most-played grouping and right-hand information. Monthly listeners, Artist Pick and biography are not in the existing artist model. Popular tracks and related artists are grant-dependent. |
| [About Artist](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=415-11733) | No About subpage | **Local + External.** Layout can be built. Biography, world rank and listeners by city lack a supported source here. Followers are optional and grant-dependent. Never relabel followers as monthly listeners. |
| [Artist albums grid](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-19098) | Artist discography and `DiscographyFilter` | **Reuse + Local.** Add persistent grid/list selection and the designed cards. Reuse paginated artist albums and EP classification. |
| [Artist albums list](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-19272) | Artist albums + collection tracks | **Local.** Add expandable album rows. Fetch tracks only on expansion, cache by album ID, and reuse table/play/queue semantics. Component `367:122923` contains closed/open variants. |
| [Podcast](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-18850) | `ui/show.rs`, `ShowPage`, `ShowEpisodes` | **Reuse + Local + External.** Recompose cover/about, follow, episode cards and filters. Resume positions and descriptions exist. Ratings/counts and podcast category labels have no source in the current Show model. Downloads/video remain separate gaps. |
| [Episode](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-18973) | `ApiRequest::Episode`, episode row playback | **Local + External.** Add dedicated detail route, description, save/share/queue and resume. Existing links open the containing podcast. The designed video surface needs a supported video backend; audio playback does not provide video. |
| [Discover](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-17144), [alternate Discover](https://www.figma.com/design/khwiuF1CnWWACfQt8RATjx/?node-id=298-17367) | Home recommendations/search-backed `Discover` request, radio | **Local + Conditional + External.** Add a separate feed route and card navigation, content/genre filters, playback and save actions. `ApiRequest::Discover` currently searches playlists; it is not Spotify's Discover feed. Exact feed ranking, video/Canvas, and engagement counters are unavailable here. A fork-owned recommendation feed is possible and must be labeled accurately. |

## 3. Mobile and responsive map

The supplied mobile frames are design references, not evidence that Spotifast already targets iOS or Android. Implementing narrow desktop layouts can reuse their hierarchy. Shipping mobile apps also requires native lifecycle, secure storage, audio/background playback, OAuth redirects, touch interactions, packaging and device testing. These are additional platform deliverables.

| Mobile design | Node(s) | Mapping and extra work |
| --- | --- | --- |
| Home, two states | `24:310`, `367:148373` | Same Home data as desktop; bottom navigation, horizontally scrolling chips/shelves, compact player and feed editor. |
| Premium profile | `261:20889` | New Profile view: existing current-user identity and history; friends, follower relationships and subscription price have separate gaps. |
| Free profile | `261:23131` | Same route with plan-aware presentation. Do not promise free-account local playback. Do not hardcode the sample price or plan name. |
| Search | `188:27427` | Search/Browse model shared with desktop; full-width input and touch tiles. |
| Library grid/list | `190:28216`, `261:22349` | Unified library state shared with desktop; filter/sort/view sheets and touch targets. |
| Playlist | `190:31599` | Existing playlist actions in a narrow layout; keep search and row menus accessible. |
| Discover, five states | `170:27706`, `176:7663`, `176:7130`, `176:7366`, `176:6894` | Shared feed and capability limits; swipe/page transitions, filter overflow and compact overlays. |
| Now Playing state board | `42:414` (430 × 3157) | Fullscreen player, queue preview, lyrics, expand/collapse and context actions. It is a board of multiple states, not one 3157-pixel screen. |
| Queue gesture state board | `238:10413` (430 × 1046) | Remove/swipe and queue transitions. Reuse only operations supported by the active playback target. |
| Shared navigation and library variants | `188:27232`, `200:11153`, `200:11326`, `200:11432`, `200:11510`, `200:11576` | Compact menu and mixed library rows. Reuse desktop models instead of a second source of truth. |

There are 13 top-level 390 × 844 mobile screens, plus the two player/gesture state boards. The design does not provide every loading, empty, offline, keyboard-focus or error state; these must be supplied using the same visual system.

## 4. Shared controls and interaction map

### Shell, navigation and library

| ID | Design/control | Current support | Work and honest behavior |
| --- | --- | --- | --- |
| S1 | Global menu `298:32652`: My Library, Home, Discover, Search | Home/Search routes exist | Add Library/Discover destinations, selection, history and narrow layout. Preserve back/forward and keyboard search access. |
| S2 | Sidebar `359:111636`, collapsed/expanded playlist group `451:18439` | Sidebar, folder collapse, pinned contexts and local playlist order exist | Recompose Pins, Playlists, Liked songs, Saves, Albums, Folders, Podcasts, Audiobooks, Artists. Clicking a category opens a filtered library, not an inert heading. |
| S3 | Pins `117:2679`, pin/unpin/move | `pinned_contexts`, `ArrangeLibrary` | **Reuse.** Local pins can match the design. Spotify app pin synchronization needs a different service. |
| S4 | Library filter `132:2985`, menus `132:2993`/`132:3017` | Type-specific views and library search | **Local.** Add All/Songs/Playlists/Albums/Artists/Folders/Podcasts/Audiobooks plus grouping. Keep filter and selection consistent across views. |
| S5 | View menu `132:3087`, `132:3096`: List/Compact/Grid | Grid preference and list/grid widgets | **Local.** Three modes backed by one result model; migrate the existing boolean preference. |
| S6 | Sort, recent ordering, library search | `LibrarySort`, `Library.filter`, play history | **Reuse + Local.** Mixed-type sorting needs explicit rules; load remaining pages when a global sort requires them. |
| S7 | Folders and nested picker | Session rootlist read, collapsed folders | **Local** folder CRUD/move is feasible using local IDs and persisted membership. **External** for Spotify-synchronized writes until proven session support exists. Never send fork folder IDs as Spotify URIs. |
| S8 | Saves bookmark versus Liked songs | Saved episodes and liked tracks are distinct | **Unspecified.** Use saved episodes for the current bookmark semantics; a separate save-for-later list for arbitrary music needs local bookmarks. The file does not define a second Spotify track-save API. |
| S9 | Bell / What's new | App release updater exists | **Local** followed-artist/show release inbox with bounded polling and read markers. Spotify's own notification feed is **External**. Keep app updates and music releases clearly separate. |
| S10 | Private-session indicator | Web API field is documented; `Device` currently omits it | **API gap** for read-only status; add optional `is_private_session`. Turning Spotify private session on/off is **External**. A local history privacy setting must not claim to hide Spotify activity. |
| S11 | Friends button / drawer | No friend activity model | **External** for Spotify activity, user search and Facebook discovery. A separate opt-in social service is a new product/backend dependency, not required to render the rest of the UI. |
| S12 | Settings and avatar/profile | Settings, current user, sign-out exist | **Reuse + Local.** New Profile route and menu; retain full existing preferences. Billing details require external account management; see G12. |
| S13 | Open in new tab menu `382:151165` | One central route/history | **Local.** Add tab state and per-tab route/filter/scroll state, or change the menu to an explicitly supported navigation action. A native tab is not a Web API feature. |

### Home and Search

| ID | Design/control | Current support | Work and honest behavior |
| --- | --- | --- | --- |
| H1 | All / Music / Podcasts / Audiobooks | Separate library entity types | **Local**, plus audiobook API work. Keep selection persistent and filter shelves without duplicating fetches. |
| H2 | Made For You / Your top mixes | Search-backed `DISCOVER_TERMS`; recommendations/top items | **Conditional.** Existing named-playlist searches do not guarantee the listener's personalized Daily Mix. Prefer known account playlists/session contexts and report unavailable results. A title match alone is insufficient evidence of personalization. |
| H3 | Favorite artists, recently played, your playlists | Top artists, history, library data | **Reuse.** Restyle and share caches. Recent searches currently store query strings, so artwork-backed recent entities need a separate small history model. |
| H4 | Albums, podcasts, episodes and audiobooks for you | Saved podcasts/newest episodes; recommendations are tracks | **Local/API gap/Conditional.** Build shelves from named sources. Saved-show episodes are not a global podcast recommendation feed. Audiobook metadata needs new models and requests. |
| H5 | Pin to Home / Hide section (`94:2649`, `39:1462`) | Two visibility booleans | **Local.** Add ordered shelf IDs, hidden state and custom library shelves. Pin-to-Home is distinct from sidebar library pins. |
| H6 | Customize feed `217:43859` / `42:371`: 11 row types, recommendation toggle, Select from Library | Only Made for You/recommendations visibility | **Local.** Persist ordered row definitions and source choices. Migrate old booleans. Disable unnecessary fetching for hidden/disabled sources where safe. |
| H7 | Browse all categories: genre, activity, entertainment, podcast and audiobook groups | No browse landing model | **Local** curated labels launching supported search. **Conditional/API gap** for Spotify categories/editorial/new-release endpoints; route only when the grant supports them. Do not label generic search results as Spotify charts. |
| H8 | Discover feed next/previous, like/save, genre filters | Search, save, play, radio primitives | **Local** feed composition and feedback. **External** exact Spotify ranking and public-looking like/play counters. Keep one active playback source when moving between cards. |

### Playback, queue, menus and lyrics

| ID | Design/control | Existing action/path | Missing work / constraint |
| --- | --- | --- | --- |
| P1 | Persistent bar `298:16739` / `113:1506` | TogglePlay, Next, Previous, Seek, SetVolume, ToggleMute, shuffle/repeat | **Reuse.** Horizontal control hierarchy, compact metadata, cover and responsive overflow. Preserve local/remote target selection and optimistic state. |
| P2 | Heart/save, add-to-playlist, share, album/artist links | ToggleSaved, AddToPlaylist, CopyLink, Open/OpenUri | **Reuse.** Correct item type, saved state and tooltips; artwork/title opens the new Song/Episode page. |
| P3 | Multi-playlist picker `104:988`, `150:10802`, `150:10866`, `150:10930`, `150:11003` | Playlist dialog, create/add/check duplicates | **Local.** Folder browsing, multi-select, staged Cancel/Done and per-playlist results. Spotify has no atomic write across several playlists: retry failed destinations without repeating successful writes. |
| P4 | Queue / Recent `94:2753`, `238:10413` | QueueTab, QueueMany, MoveInQueue, InsertInQueue, ClearQueue, history | **Reuse + Local.** Redesign presentation. Current local queue can reorder; the remote Web API supports read/append, not arbitrary remove/reorder. Check the active target before enabling gestures/actions. |
| P5 | Auto-Play Similar Tracks switch | `settings.autoplay`, session context resolver | **Reuse + Conditional.** This is local-player autoplay, not a new global Spotify account preference. |
| P6 | Lyrics panel `98:3017`; fullscreen player | `ui/lyrics.rs`, Spotify session lyrics + LRCLIB fallback | **Reuse.** Sync, seek, follow/pause-follow, fullscreen, loading and unavailable states. For a Song page showing a different song, do not silently show current-playing lyrics; key queries by selected song. |
| P7 | Annotation/Genius lyric states `359:114584` | No annotation provider/model | **External.** Add licensed provider integration or an external attribution link. Do not scrape or embed the Figma sample annotation as real data. |
| P8 | Sleep timer in track menu | No sleep-timer action or deadline | **Local.** Runtime timer that issues explicit Pause to the selected target, supports cancel and end-of-track, survives hidden-window operation and cannot resume a paused player. Define behavior after device transfer. |
| P9 | Hide song in feed/menu | No hidden-track preference | **Local.** Persist exclusions for fork feed/autoplay only; distinguish from removing a track from a playlist. Spotify-wide recommendation feedback is **External**. |
| P10 | Download song/episode icons | Audio cache only | **External.** The cache is not a supported offline download library. Do not expose a successful Download state based on cached audio. |
| P11 | Device picker | Devices/Transfer, local receiver discovery | **Reuse.** Surface restricted/unavailable targets. Starting an unwoken Google Cast receiver is separate missing sender work. |
| P12 | Miniplayer / Fullscreen menu | Winamp mini player, fullscreen lyrics | **Reuse + Local.** Preserve current mini player and add the designed fullscreen Now Playing composition. Fullscreen lyrics alone does not fulfill that screen. |
| P13 | DJ component `181:11652`, open/hover states | Radio and autoplay; no Spotify DJ integration | **External** for Spotify DJ voice, selections and feedback. A local radio-feedback widget is implementable but must have its own truthful name and semantics. |
| P14 | Radio, similar tracks | OpenSongRadio, Radio page, session stations; conditional recommendations | **Reuse + Conditional.** Song's More like this can use an identified available source. Do not claim it exactly matches Spotify's recommendation ranking. |

### Detail pages and social state boards

| ID | Design/control | Support and additions |
| --- | --- | --- |
| D1 | Artist tabs `161:4629`: Home, Albums, Singles and EPs, Compilations, Merch, About, Features & More | Discography filters exist. Tab routing/state is **Local**. Merch is **External/Unspecified**, with no supplied detailed screen. Appears-on releases can support part of Features & More; editorial content cannot be inferred from them. |
| D2 | Your most played on artist page | **Local** aggregation of the listener's history/top tracks. Label its time window and coverage; it is not global play count. |
| D3 | Song Credits tab and contributor chips | Primary performing artists, album label/copyright and some metadata exist. Composer, lyricist, producer and full personnel require an added source. Do not promote all `Track.artists` into composers. |
| D4 | Artist banner, Pick, bio and detailed statistics | Image assets can be adapted from supplied artist metadata; the intended banner crop and artist-selected Pick are not equivalent to an ordinary artist image/album. Missing fields are optional and need provider provenance. |
| D5 | Podcast follow, All Episodes / New, save and resume | Existing follow/save and resume data support most actions. Define New from release time/read state, not an invented Spotify notification flag. Full played/unplayed filter behavior needs an explicit model and tests. |
| D6 | Podcast 4.8 rating / 816K count | Neither value exists in the current Show model or inspected public Show schema. Hide unavailable statistics; don't infer them from follower/episode counts. |
| D7 | Episode video surface and video badge | Current playback supplies audio; image artwork is not a video stream. Badge should only claim video when metadata proves it. A browser link to Spotify can remain available. |
| D8 | Friends board `213:11558`: activity, add/search, Facebook, follow confirmation | Six depicted states. All need real identity/activity services. Public user lookup (where granted) is not a Spotify-wide user search or friends activity API. |
| D9 | Profile followers/following/public playlists | Current User model lacks these fields. Some legacy/extended endpoints expose limited profile data; newer app restrictions apply. Counts must remain optional. Friends and following are not interchangeable. |

## 5. Missing supporting APIs and data, with concrete paths forward

This register covers absent services, absent client integrations, and easily confused data. **A missing client method is not a missing Spotify API.** Existing endpoint wrappers remain in `src/api/client.rs`; they should be reused rather than duplicated.

| Gap | Missing contract | Implementation path | Fallback / release condition |
| --- | --- | --- | --- |
| G1 Audiobook catalogue | `Audiobook`, `Chapter`, saved-book collection, search results, request/response/cache types | Add `GET /audiobooks/{id}`, `/audiobooks/{id}/chapters`, `/chapters/{id}`, `/me/audiobooks` and audiobook search. Reuse library save/unsave/contains URI operations where accepted. Handle author/narrator and market restrictions. See [audiobook API][audiobook]. | Browse/save can ship independently of playback. Existing saved-show audiobook detection is not a full audiobook catalogue. |
| G2 Audiobook playback | No chapter support in current `PlayableItem`/player workflow | First prove a supported playback path. Keep catalogue models distinct from playable tracks/episodes; arbitrary chapter URIs cannot simply be cast to episodes. Remote control is also unverified, not an assumed workaround. | Open the title in Spotify. Do not claim local play, resume or download without an end-to-end proof. |
| G3 Private-session status | `Device.is_private_session` omitted | Add optional bool, preserve unknown/false/true, render read-only status from the current playback/device response. [Documented field][playback]. | No public toggle identified. Omitted field is unknown, not false. |
| G4 Exact Home / Discover / Daily Mix | No personalized Home feed contract; current Discover request is playlist search | Add a fork feed built from known saved playlists, recent/top items and supported station/recommendation results, with source IDs and ordered rows. | Hide or explain unavailable shelves. Never select a random same-named playlist and present it as the listener's Daily Mix. |
| G5 Genre browse, charts, new releases | No browse API wrappers/models in this fork; some endpoints grant-restricted | Curated navigation can launch search now. Optional browse/editorial requests need capability checks. A releases feed can scan followed artists at bounded intervals. | Curated search and followed-artist releases must be named accordingly, not sold as Spotify charts/global editorial ranking. |
| G6 Track counts, artist listeners/rank/cities | No corresponding track/artist fields or supported service here | Keep global plays, monthly listeners, follower count, popularity and local play count as separate optional values with provenance. [Artist schema][artist], existing model and session reads were inspected. | Omit absent metrics. Existing artist followers may be absent for newer Development Mode apps. |
| G7 Bio, Artist Pick, merch, richer genres and credits | Missing source and metadata types | Add an explicit provider boundary if a supported licensed source is selected. Basic artist names/album copyrights use current models. Artist genres cannot automatically be labeled track genres. | Show only known metadata and external links. No hardcoded sample bio, roles, tags or artist endorsements. |
| G8 Podcast ratings, rating count and categories | Missing in current Show model and inspected [Show API][show] | New provider/support proof needed for each field. Description, publisher when present, images, episode count and episodes already have paths. | Show About and episode list without invented rating UI. |
| G9 Genius annotations | Missing authorization/provider, annotations, ranges, attribution and caching | Separate optional lyrics-annotation integration; keep Spotify/LRCLIB lyric timing independent. Validate permitted display/caching before bundling text. | Existing lyrics still work; external Genius navigation can be separate. |
| G10 Spotify social graph/activity/Facebook | No friend-activity feed, user-search or Facebook-friend discovery integration | A separate opt-in social service would need identity, consent, presence, blocking, data retention and a hosted backend. That is a product extension, not a skin feature. | Do not show mock friends as connected users. Hide entry or provide an honest unavailable state. |
| G11 Spotify DJ, feedback and Smart Shuffle | No service in public client API or pinned integration | Existing radio/autoplay supports a differently named local experience. Exact Spotify DJ requires supported service/player integration. | Keep unavailable Spotify-specific controls out of the working player flow. |
| G12 Billing plan detail and price | Current user product may be available; no billing price/plan-family contract | Use verified account tier only when present. Open Spotify account management for billing. | Do not infer Individual/Family or regional price from Premium. Figma's dollar amount is sample content. |
| G13 Spotify folder writes and pin synchronization | Rootlist is read-only; no public folder/pin write API | Implement local folders, membership, order and pins with account-scoped persistence. Session write support would need a separate proven adapter and tests. | Distinguish local organization from Spotify-synchronized organization. Retain existing rootlist import. |
| G14 Remote queue delete/reorder | Public API exposes queue read/add, not arbitrary edits | Keep current local-player mutations. For remote targets expose only supported actions. Do not reconstruct a remote queue by starting a shortened track list. | Disable/remove unsupported queue edit controls with a reason, preserving remote playback. |
| G15 Video podcasts / Canvas | No video stream/decoder or Canvas service in the audited path | Separate provider, authorization, decoder, rendering and sync work. Audio artwork is insufficient. | Audio episode view plus Spotify link; no fake video player. |
| G16 Offline downloads | No supported offline entitlement/download lifecycle | Requires lawful player support including availability and entitlement refresh. Current playback cache cannot fulfill the contract. | Download actions unavailable; cached audio is not promised as offline listening. |
| G17 Sleep timer, hide, bookmarks, tabs | No corresponding local state/actions | Add runtime deadline/cancel action, local exclusion/bookmark models and tab state. No new Spotify endpoint required. | Define device-transfer, sign-out, restart and privacy behavior; keep settings atomic/backward compatible. |
| G18 Playback speed, crossfade, local files | Outside current player features, not a metadata API gap | Separate audio-engine work: time stretching; dual-decoder/mixer transitions; file import/decoder/path lifecycle. Preserve queue, gapless, EQ and visualization contracts. | These are possible fork projects, not part of merely matching colors. They are not required by a detailed screen in this file. |
| G19 Google Cast activation | No Cast sender to wake receiver | Add a supported discovery/launch integration independently of normal Connect controls. | Already active devices may appear via Spotify's device list. Do not promise discovery of all Cast speakers. |
| G20 Free playback and lossless | Current librespot playback requires Premium; no lawful lossless stream support established | Wait for demonstrated lawful player support. A change of UI or metadata API does not alter playback entitlement. | Honest account/format state. No alternate audio substitution or DRM bypass. |

### API access is a capability, not a global boolean

The current gateway distinguishes shared and personal grants, playlist ownership, and session reads. Preserve that design. Requests classified as `UnsupportedDevelopmentMode` stay on shared access; playlist content can use session reads when supported. A successfully signed-in personal app does not imply editorial, recommendations, artist-top-track or social access.

Spotify's [2024 changes][api2024] restrict recommendations, related artists and other discovery functionality for newer use cases. The [February 2026 migration][migration] narrows newer Development Mode access, changes search/page behavior, removes several fields and changes library/playlist routes. Spotify later [postponed endpoint changes for existing integrations][api2026]. The [July update][quota] raises the client-ID allowance to 25 and shares quota across a developer's IDs. Do not copy the older one-client-ID guidance into this fork.

Several migrations are already implemented: generic library writes/contains, newer playlist item routes, different search limits, and separate quota-exhaustion handling. Audit and extend them; do not reimplement them from the Figma labels. `is_quota_exhausted` already recognizes `QUOTA_EXCEEDED`.

For new optional features, represent availability with enough detail to distinguish supported, missing grant, account/market/device restriction, temporarily unavailable, and unsupported. An HTTP error must not turn into a permanent empty library. Unknown availability must not become a fabricated count or a silently enabled play button.

### Proposed new client contracts

These names are implementation proposals, not existing APIs or committed code.

| Contract | Contents | Owner |
| --- | --- | --- |
| `LibraryViewState` | Entity-type filter, query, sort, grouping, List/Compact/Grid, selection | `model.rs`, UI; persist stable preferences |
| `SongPage`, `EpisodePage` | Selected ID, known summary, load/error state, tab, optional enrichment | `model.rs`, app route loader; reuse Track/Episode requests |
| `ArtistTab` | Home/discography/About and supported content subsets | Artist UI/model; retain pagination per filter |
| `FeedSettings` / shelf definitions | Stable shelf IDs, source, order, visibility, recommendation flag | Settings with migration from `HomeSettings` |
| `FeedItem` | Real entity URI, source/provenance, supported actions, optional metadata | Model + async feed assembly; not a fake Spotify endpoint |
| `LocalFolder`, `Bookmark`, hidden-track store | Local identity, member URIs, account namespace, timestamps | Persisted local state; no Spotify folder mutation claim |
| `Audiobook`, `Chapter`, API variants | Metadata, paging, restrictions, save status; separate from playback support | `api/models.rs`, client/backend/model/UI |
| Optional enrichment | Credits, bio, statistics, annotations, provenance and expiry | Separate provider boundary; absent by default |
| `SleepTimer` | Deadline or end-of-item mode, selected target, cancel state | Runtime/app actions; explicit pause, no UI-thread sleep |

## 6. Visual implementation contract

The inspected desktop Home reference supplies these anchors. Other pages have their own artwork tints; this is not a directive to make every surface the same color.

| Element | Figma reference | Native implementation implication |
| --- | --- | --- |
| Main background | `#060606` | Semantic window token |
| Primary/secondary surfaces | `#111111`, `#202020` | Separate selected/background/outline roles |
| Text | `#E0E0E0`, secondary `#898989` | Preserve contrast and hover/focus states |
| Accent | `#1ED760` | Existing Spotify accent can be reused |
| Typography | Satoshi, regular 14px; secondary 12px; tracking 3% in inspected components | Bundle verified licensed font assets and map weights/metrics; current Inter is not an exact match |
| Top navigation | x7/y3, 1754 × 54 in 1771px frame | Global header above sidebar/content; responsive constraints rather than fixed screen coordinates |
| Sidebar | x7/y60, 259 × 957 | Persistent library column; scrollable groups and collapsed mode |
| Main panel | x273/y60, 1491 × 869, about 10px corners | Separate bordered scrollable content area |
| Player | x273/y933, 1491 × 81, about 10px corners | Player spans content width to the right of sidebar; existing player spans full window |
| Home playlist cards | 170px artwork, about 11px horizontal gap, stacked covers | Extend existing 172px cards, whose internal cover is smaller; preserve dynamic API artwork |
| Shared icon assets | `14:11`, `24:316` | Export the exact supplied static assets; don't approximate them with unrelated icons and claim pixel fidelity |

Read full design context for each implementation subtree before editing. The metadata inventory alone is not a visual implementation spec. The audit has inspected high-fidelity shell, sidebar, player, feed/card, DJ, friends, song-tab and podcast-row context, plus the overall Home reference. Detailed implementation needs equivalent context for remaining screen subtrees.

The Figma file supplies dark designs. Light mode is an adaptation using the same geometry and existing semantic palette behavior, not a supplied pixel-perfect light reference. At narrow desktop widths, collapse side panels and prioritize player controls instead of shrinking text or creating overlapping fixed-position regions. Exact breakpoints should be derived from measured content and tested at 1771×1024, 1440×900 and 900×700; these latter sizes are proposed QA sizes, not Figma frames.

Static design asset delivery is still an implementation dependency: temporary Figma asset URLs must not ship in code. Direct asset downloads did not succeed in this workspace during the audit. Sample album art, account names, prices, lyrics and statistics are reference data, not production fixtures to show after real sign-in.

## 7. Implementation work order and completion gates

Each milestone below is currently **not implemented** by this audit. Complete a working vertical slice rather than adding a field of nonfunctional buttons.

| Milestone | Concrete scope | Dependencies and acceptance |
| --- | --- | --- |
| M1 Native shell and design foundation | Tokens, licensed font/icons, global nav, sidebar, bordered content, player composition and responsive behavior | Existing playback actions continue to work. Capture before/after dark/light, normal/narrow and open overlays. No pixel-fidelity claim before asset/geometry checks. |
| M2 Unified library and Home | Library filter/sort/views, feed settings, local pins/folders/bookmarks, Home chips and shelves, Search browse landing | Settings migration, pagination, meaningful empty/error states, real data provenance, no bogus personalized mixes. |
| M3 Music detail pages | Album/playlist redesign, Song route and tabs, artist tabs and expandable discography | Reuse existing cache/actions; permissions and optimistic edits retained; missing credits/stats remain optional. |
| M4 Podcasts and player surfaces | Podcast/Episode detail, lyrics drawer/fullscreen Now Playing, queue/recent, multi-playlist picker, timer | Resume point correctness, lyrics scoped to selected track, active-target queue constraints, picker partial-failure handling. |
| M5 Discover and account surfaces | Fork feed, Profile, private-session status, release inbox | Feed source attribution, read-only private status, accurate account/billing presentation, conditional recommendations. |
| M6 Audiobook catalogue | Search/library/detail/chapter metadata/save and Spotify handoff | Account/market-aware metadata; no unsupported playback/download claim. |
| M7 Optional services/platforms | Supported enrichment, exact Spotify-only services if support is proven, native mobile port if pursued | Each needs its own service contract and end-to-end evidence. UI-only completion cannot mark this milestone complete. |

### Required behavior checks

- All Figma navigation targets implemented for the native desktop scope resolve to a real page, drawer or explicit unavailable state. Label-only destinations are specified before being called complete.
- Tests cover new route persistence, old-settings migration, local folders/pins, feed ordering/filtering, and account separation. Use current regression suites for queue and playlist behavior rather than replacing them.
- Test local playback and remote Connect separately, including permissions, unavailable items, pause/resume, seek, duplicate queue entries, device transfer and sign-out while a request is pending.
- No unsupported action reports success. Saves, multi-playlist writes and reorders remain optimistic, but partial errors retain accepted changes and expose recovery.
- Optional metadata absent from newer grants stays absent. Never display `0` for unknown global counts or treat a missing private-session flag as false.
- UI remains usable with no saved content, failed artwork, unavailable lyrics, API 403, HTTP 429, exhausted quota, offline state and paginated loading.
- Keyboard navigation, focus, accessibility labels, long titles, text scaling and RTL remain usable. API work stays off the UI thread; hidden views do not start unbounded polling.
- Run the checks in [CONTRIBUTING.md](CONTRIBUTING.md#checks), and produce its HTML visual comparison for actual interface changes. Report real platform coverage. No visual code has changed in this mapping commit.

## 8. Decisions that the design leaves open

These do not block shell or existing-feature implementation. They need explicit definitions before their individual features are considered complete.

1. **Saves:** saved podcast episodes alone, or a fork-local read/listen-later collection across content types? Proposed default: existing saved episodes plus an explicitly local bookmark collection if broader saving is desired.
2. **Organization:** local editable folders/pins can ship now. Spotify synchronization remains a separate integration, not an implied promise.
3. **Discover:** proposed implementation uses real supported sources in a fork-owned feed. Exact Spotify personalization is a different capability.
4. **Unsupported destinations:** preserve discoverability with clear unavailable/detail states where useful; do not leave controls that animate but cannot act. Merch, notification details and several tab bodies lack complete Figma screens.
5. **Mobile:** use narrow-layout inspiration for the desktop fork first. A fully packaged iOS/Android product needs a separate platform implementation and verification plan.
6. **Provider additions:** biographies, personnel credits, annotations and social activity require selecting supported providers and documenting requests/storage. No external provider or hosted service has been added by this audit.

## 9. Evidence and maintenance

Inspected the supplied Redesign page's complete top-level metadata, screen content labels, key component variants and selected prototype links; inspected source routes, actions, backend requests, models, settings and relevant UI modules. The inventory distinguishes screen frames from component/state boards. The mapping is a static audit and plan, not a playback or visual QA pass.

Mapping validation passed for all 114 top-level inventory entries, the 16 desktop/13 mobile screen totals, linked Figma node IDs, local source links and feature/gap identifiers. `git diff --check` and `cargo fmt --all --check` passed. The launcher checks had two errors because Ruby is missing, and the Jekyll build could not start because Bundler is missing. An attempted baseline demo build failed while creating the `syn` archive with a memory-map error; no native screenshots or full Rust test pass were obtained. These environment limits do not establish an application regression, but native build and visual verification remain open implementation gates.

Relevant repository contracts: [How It Connects](docs/_reference/how-it-connects.md), [Queue Rules](docs/_reference/queue.md), [What Spotify Allows](docs/_reference/what-spotify-allows.md), [theme format](docs/_reference/settings-and-files.md), [contribution policy](CONTRIBUTING.md).

Review this map when the pinned player, grant restrictions, supplied Figma file or source routes change. Update implemented status only after the relevant native behavior and evidence exist. Do not interpret an enum/protobuf name or a successful mock as proof of Spotify service support.

[api2024]: https://developer.spotify.com/blog/2024-11-27-changes-to-the-web-api
[api2026]: https://developer.spotify.com/blog/2026-02-06-update-on-developer-access-and-platform-security
[migration]: https://developer.spotify.com/documentation/web-api/references/changes/february-2026
[quota]: https://developer.spotify.com/blog/2026-07-23-web-api-quota-updates
[playback]: https://developer.spotify.com/documentation/web-api/reference/get-information-about-the-users-current-playback
[audiobook]: https://developer.spotify.com/documentation/web-api/reference/get-an-audiobook
[artist]: https://developer.spotify.com/documentation/web-api/reference/get-an-artist
[show]: https://developer.spotify.com/documentation/web-api/reference/get-a-show
