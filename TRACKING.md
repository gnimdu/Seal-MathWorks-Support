# MathWorks Video Support - Implementation Tracking

## Status: IN PROGRESS

| # | Task | Status | Notes |
|---|------|--------|-------|
| **Phase 1: URL Resolver Core** | | | |
| 1.1 | Create `UrlResolver.kt` with MathWorks detection & resolution | [ ] TODO | New file in `util/` |
| 1.2 | JSON-LD parsing & Brightcove URL extraction | [ ] TODO | Part of UrlResolver |
| **Phase 2: Integration** | | | |
| 2.1 | Integrate resolver into `DownloadUtil` (3 methods) | [ ] TODO | fetchVideoInfo, getPlaylistOrVideoInfo, downloadVideo |
| 2.2 | Verify Task/DownloaderV2 compatibility | [ ] TODO | Ensure resolved URLs flow correctly |
| **Phase 3: Testing** | | | |
| 3.1 | Unit tests for UrlResolver | [ ] TODO | URL matching, JSON-LD parsing |
| 3.2 | Integration test with real MathWorks URL | [ ] TODO | End-to-end via yt-dlp |
| 3.3 | Edge case testing | [ ] TODO | Locales, auth, missing videos |
| **Phase 4: Polish** | | | |
| 4.1 | Error handling & logging | [ ] TODO | Graceful fallback |
| 4.2 | Documentation | [ ] TODO | Code comments |

## Key Decisions

- **Approach**: URL interception in Seal (Kotlin-side), not yt-dlp plugin
- **Why**: MathWorks uses Brightcove; yt-dlp already supports Brightcove but can't detect it on MathWorks pages (JS-loaded player, no static HTML embeds)
- **How**: Fetch MathWorks page → parse JSON-LD → extract Brightcove embed URL → pass to yt-dlp

## Test URLs

- `https://fr.mathworks.com/videos/deep-learning-for-engineers-part-1-why-choose-deep-learning-1617287683265.html`
- Expected Brightcove embed: `https://players.brightcove.net/62009828001/default_default/index.html?videoId=6245925599001`
