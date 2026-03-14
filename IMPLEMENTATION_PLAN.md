# Seal - MathWorks Video Download Support: Implementation Plan

## Context

Seal is an Android app wrapping yt-dlp for video/audio downloads. MathWorks hosts videos
via **Brightcove** (account `62009828001`), but yt-dlp's generic extractor cannot auto-detect
Brightcove on MathWorks pages because the player is loaded dynamically via JavaScript — no
static `<iframe>`, `<video>`, or `<video-js>` tags exist in the server-rendered HTML.

The Brightcove embed URL **is** available in the page's JSON-LD structured data (`embedUrl` field).
If that URL is passed directly to yt-dlp, the existing `BrightcoveNewIE` extractor handles it.

## Strategy

**Approach: URL Interception in Seal (Kotlin-side)**

Add a URL resolver layer that intercepts MathWorks URLs before they reach yt-dlp, fetches the
page HTML, extracts the Brightcove embed URL from JSON-LD, and substitutes it. This requires
no changes to yt-dlp itself and works with the existing `youtubedl-android` library.

### Why This Approach

| Approach | Pros | Cons |
|----------|------|------|
| **A) URL interception in Seal** | No yt-dlp changes, fast to ship, self-contained | Seal-specific, needs maintenance if MathWorks changes page structure |
| B) yt-dlp plugin | Proper separation of concerns | `youtubedl-android` doesn't support loading external plugins |
| C) Upstream yt-dlp PR | Benefits all users | Long review cycle, out of our control |

**Decision**: Start with **Approach A**. Optionally pursue C as a follow-up.

---

## Architecture

### New Components

```
com.junkfood.seal.util.UrlResolver.kt   (NEW)
  └── MathWorksResolver                  (URL → Brightcove embed URL)
```

### Integration Points

The resolver will be called at the **earliest URL processing stage**, before URLs are passed
to `YoutubeDLRequest`. Two main entry points:

1. **`DownloadUtil.fetchVideoInfoFromUrl()`** (line 140) — info fetching
2. **`DownloadUtil.getPlaylistOrVideoInfo()`** (line 87) — playlist/video detection
3. **`DownloadUtil.downloadVideo()`** (line 660) — actual download

All three construct `YoutubeDLRequest(url)`. The resolver transforms the URL before this call.

### Data Flow

```
User pastes: https://fr.mathworks.com/videos/deep-learning-...html
                          │
                    UrlResolver.resolve(url)
                          │
                    ┌──────────────┐
                    │ Is MathWorks?│──No──→ return original URL
                    └──────┬───────┘
                           │ Yes
                    Fetch page HTML (OkHttp)
                           │
                    Parse JSON-LD from <script type="application/ld+json">
                           │
                    Extract embedUrl (players.brightcove.net/...)
                           │
                    Return Brightcove embed URL
                           │
                    YoutubeDLRequest(resolvedUrl)
                           │
                    yt-dlp BrightcoveNewIE handles it ✓
```

---

## Implementation Tasks

### Phase 1: URL Resolver Core

**Task 1.1: Create `UrlResolver.kt`**
- File: `app/src/main/java/com/junkfood/seal/util/UrlResolver.kt`
- Implement `UrlResolver` object with:
  - `suspend fun resolve(url: String): String` — main entry point
  - `private fun isMathWorksUrl(url: String): Boolean` — regex check for `mathworks.com/videos/`
  - `private suspend fun resolveMathWorks(url: String): String` — fetch page, parse JSON-LD, return Brightcove URL
- Use OkHttp (already a dependency) for HTTP requests
- Use `kotlinx.serialization` (already a dependency) for JSON-LD parsing
- Regex pattern: `https?://(?:\w+\.)?mathworks\.com/videos/.+`

**Task 1.2: JSON-LD Parsing**
- Parse `<script type="application/ld+json">` content from HTML
- Extract `embedUrl` from the `VideoObject` typed entry
- Expected format: `https://players.brightcove.net/62009828001/default_default/index.html?videoId=XXXXXX`
- Fallback: if JSON-LD missing, try regex extraction of `data-video-id` and `data-account` attributes

### Phase 2: Integration into Download Flow

**Task 2.1: Integrate resolver into `DownloadUtil`**
- Modify `fetchVideoInfoFromUrl()` to call `UrlResolver.resolve(url)` before creating `YoutubeDLRequest`
- Modify `getPlaylistOrVideoInfo()` similarly
- Modify `downloadVideo()` — resolve URL at line 672-678 before passing to `YoutubeDLRequest`
- All three methods are already `suspend` or run in coroutine context, so the async HTTP call fits naturally

**Task 2.2: Integrate resolver into `Task` creation**
- Option: resolve URL in `TaskFactory` before `Task` construction
- Alternative: resolve in `DownloaderV2Impl` when task starts processing
- Preferred: resolve at the `DownloadUtil` level (Task 2.1) to keep Task as a simple data class

### Phase 3: Testing & Validation

**Task 3.1: Unit Tests**
- Test `UrlResolver.isMathWorksUrl()` with various URL formats:
  - `https://www.mathworks.com/videos/...`
  - `https://fr.mathworks.com/videos/...`
  - `https://mathworks.com/videos/...`
  - Non-MathWorks URLs (should pass through unchanged)
- Test JSON-LD parsing with sample HTML
- Test fallback behavior when JSON-LD is missing

**Task 3.2: Integration Testing**
- Test with real MathWorks URL: `https://fr.mathworks.com/videos/deep-learning-for-engineers-part-1-why-choose-deep-learning-1617287683265.html`
- Verify the resolved Brightcove URL works with yt-dlp
- Test on Android emulator via Android Studio

**Task 3.3: Edge Cases**
- MathWorks URL with no video (e.g., product page) — should fail gracefully
- Network errors during page fetch — should return original URL or clear error
- Different MathWorks locale subdomains (fr., de., jp., etc.)
- Videos that may require authentication

### Phase 4: Polish

**Task 4.1: Error Handling**
- Graceful fallback: if resolver fails, pass original URL to yt-dlp (may produce a clearer error)
- Log resolver activity for debugging
- User-facing error message if MathWorks page structure changes

**Task 4.2: Documentation**
- Add MathWorks to any user-facing list of supported sites (if one exists)
- Code comments explaining the Brightcove workaround

---

## Key Files Reference

| File | Role |
|------|------|
| `app/src/main/java/com/junkfood/seal/util/DownloadUtil.kt` | Core download logic, URL → yt-dlp |
| `app/src/main/java/com/junkfood/seal/util/VideoInfo.kt` | Metadata models |
| `app/src/main/java/com/junkfood/seal/util/TextUtil.kt` | URL matching utilities |
| `app/src/main/java/com/junkfood/seal/download/Task.kt` | Download task data class |
| `app/src/main/java/com/junkfood/seal/download/DownloaderV2.kt` | Task orchestration |
| `app/src/main/java/com/junkfood/seal/download/TaskFactory.kt` | Task creation |
| `gradle/libs.versions.toml` | Dependency versions |

## Dependencies (already in project)

- **OkHttp 5.0.0-alpha.10** — HTTP client for fetching MathWorks pages
- **kotlinx.serialization** — JSON parsing for JSON-LD data
- **youtubedl-android 0.17.3** — yt-dlp wrapper (BrightcoveNewIE handles resolved URLs)

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| MathWorks changes page structure | JSON-LD is a web standard, unlikely to be removed; add fallback regex |
| Brightcove account ID changes | Extract dynamically from JSON-LD, don't hardcode |
| Some videos require auth | Support cookie passthrough (already exists in Seal) |
| Network latency from extra HTTP request | Resolver runs before yt-dlp fetch, negligible overhead |
