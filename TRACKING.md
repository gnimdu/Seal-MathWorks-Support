# MathWorks Video Support - Implementation Tracking

## Status: COMPLETE

| # | Task | Status | Notes |
|---|------|--------|-------|
| **Phase 1: URL Resolver Core** | | | |
| 1.1 | Create `UrlResolver.kt` with MathWorks detection & resolution | [x] DONE | `app/src/main/java/com/junkfood/seal/util/UrlResolver.kt` |
| 1.2 | JSON-LD parsing & Brightcove URL extraction | [x] DONE | Two strategies: JSON-LD + HTML fallback |
| **Phase 2: Integration** | | | |
| 2.1 | Integrate resolver into `DownloadUtil` (3 methods) | [x] DONE | getPlaylistOrVideoInfo, fetchVideoInfoFromUrl, downloadVideo |
| 2.2 | Verify Task/DownloaderV2 compatibility | [x] DONE | Task is data class, resolver runs at DownloadUtil level |
| **Phase 3: Testing** | | | |
| 3.1 | Unit tests for UrlResolver | [x] DONE | 8 tests passing (URL matching, JSON-LD parsing, fallback) |
| 3.2 | Live validation with real MathWorks URL | [x] DONE | Confirmed JSON-LD VideoObject with Brightcove embedUrl exists |
| 3.3 | Integration test on Android emulator | [x] DONE | 2 instrumented tests passing on emulator (Pixel 3a API 34) |
| **Phase 4: Polish** | | | |
| 4.1 | Error handling & logging | [x] DONE | Graceful fallback to original URL on failure; fixed HTTP 403 with proper headers |
| 4.2 | Documentation | [x] DONE | Code comments in UrlResolver.kt |

## Key Decisions

- **Approach**: URL interception in Seal (Kotlin-side), not yt-dlp plugin
- **Why**: MathWorks uses Brightcove; yt-dlp already supports Brightcove but can't detect it on MathWorks pages (JS-loaded player, no static HTML embeds)
- **How**: Fetch MathWorks page → parse JSON-LD → extract Brightcove embed URL → pass to yt-dlp

## Verified End-to-End on Emulator

```
D UrlResolver: Resolving MathWorks URL: https://fr.mathworks.com/videos/deep-learning-...html
D UrlResolver: Resolved MathWorks → Brightcove: https://players.brightcove.net/62009828001/default_default/index.html?videoId=6245925599001
```

## Test URLs

- `https://fr.mathworks.com/videos/deep-learning-for-engineers-part-1-why-choose-deep-learning-1617287683265.html`
- Expected Brightcove embed: `https://players.brightcove.net/62009828001/default_default/index.html?videoId=6245925599001`

## Files Modified/Created

- **NEW**: `app/src/main/java/com/junkfood/seal/util/UrlResolver.kt` - URL resolution logic
- **NEW**: `app/src/test/java/com/junkfood/seal/UrlResolverTest.kt` - Unit tests (8 tests)
- **NEW**: `app/src/androidTest/java/com/junkfood/seal/UrlResolverIntegrationTest.kt` - Instrumented tests (2 tests)
- **MODIFIED**: `app/src/main/java/com/junkfood/seal/util/DownloadUtil.kt` - Integration (3 methods)

## Git History

- `e6b638b2` - Add MathWorks video download support via URL resolution
- `9108ca6d` - Fix HTTP 403 from MathWorks and add integration test
