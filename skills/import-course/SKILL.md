---
name: import-course
version: 0.3.0
description: Import an online course from any platform (Kajabi, Skool, Teachable, WordPress/LearnDash, etc.) into Simplero. Handles video, audio, text, attachments, and resources.
user-invocable: true
argument-hint: <source-url>
---

# Import Course into Simplero

You import online courses from external platforms into Simplero. You handle the entire pipeline: scraping the source, downloading media, uploading assets, and creating the lesson structure.

## Step 1: Gather Information

Ask the user for anything you don't already have:

1. **Source URL** — the course/classroom URL on the source platform
2. **Source credentials** — email/password for the source platform
3. **Simplero API key** — header name is `X-API-Key` (not `X-Simplero-API-Key`)
4. **Simplero target** — where to put the content. Ask: "Which Simplero course and module should I import into?" Then use `GET /courses.json` to list courses, or `GET /courses/{id}.json` to show modules in a specific course. Help them pick or create the right target.

## Step 2: Understand the Simplero Structure

Simplero's course hierarchy:

```
Site
└── Course
    └── Module (a section/grouping)
        └── Lesson
            ├── title (string)
            ├── body (HTML — the lesson's text/copy content)
            ├── asset_id (a single video or audio file)
            ├── attachments[] (PDFs, downloads, links)
            ├── publish_status ("published" or "draft")
            ├── default_playback_speed (float, e.g. 1.0 or 1.5)
            ├── allow_multiple_completions (boolean)
            └── remember_playback_position (boolean)
```

**Mapping rules:**
- If the source has a flat list of lessons → import into a single Simplero module
- If the source has sections/categories/sub-modules → each becomes a Simplero module (ask user first, or create modules via `POST /courses/{course_id}/course_modules.json` with `{"title": "...", "publish_status": "published"}`)
- Every video/audio → one Simplero lesson with the file as the `asset_id`
- Text/HTML content from the lesson page → lesson `body`
- PDFs, downloads, Google Drive files → lesson `attachments` (download the actual file when possible, don't just link)
- External links that can't be downloaded → link-type attachments
- SoundCloud/audio embeds → download as mp3 and upload as the lesson asset

## Step 3: Scrape the Source Platform

Use Playwright (headless Chromium) to log in and scrape. Key patterns per platform:

### General Approach
1. Launch browser with anti-detection: `headless=True`, custom user-agent, remove webdriver flag
2. Log in via the platform's login form
3. Navigate to the course page
4. Extract lesson list with titles and URLs — **check for pagination, "Show More" buttons, and "Load More" triggers**. Many platforms only show 10-20 lessons at a time. Always look for:
   - `?page=N` pagination links
   - "Show More" / "Load More" buttons (may reveal pagination after clicking)
   - Infinite scroll (scroll to bottom, wait, check for new items)
   - A total lesson count displayed on the page — compare against how many you've found
5. Visit each lesson page to get: video URLs, body HTML, download links, audio embeds

### Platform-Specific Notes

**Skool:**
- After login, go to `/GROUP/classroom`
- Read `window.__NEXT_DATA__` for `buildId` and `allCourses` (in `pageProps`)
- Course titles are in `metadata.title`, not `name`
- Navigate to each course URL to get `children` — each child has `course.metadata` with `title`, `videoStream` (BunnyCDN HLS), `resources` (JSON string of links)
- Children with `unitType: "set"` are sub-modules — recurse into their `children`
- Children with `unitType: "module"` are individual lessons

**Kajabi:**
- Login at `/login` — fields are `#member_email` and `#member_password`. Submit button may be `input[type="submit"]` OR `button#form-button` — check the page.
- Course page lists categories with links to `/categories/{id}`
- Each category page has lesson links to `/posts/{id}`
- Videos are typically Wistia embeds — look for `wistia_async_{ID}` in div classes
- **Pagination and "Show More" (CRITICAL):** Kajabi paginates lesson lists. You MUST handle this:
  1. The course listing page (`/products/{slug}`) may show only the first 10 lessons with a **"Show More"** link at the bottom. Click it.
  2. After clicking "Show More", **check for pagination links** (`?page=2`, `?page=3`, etc.) that appear at the bottom. These are NOT visible until after Show More is clicked.
  3. Also check `?page=N` on the course URL itself — some courses paginate at the top level (e.g., `/products/{slug}?page=2`).
  4. For category pages (`/categories/{id}`), also check for `?page=N` pagination.
  5. **Always verify the total lesson count** — Kajabi shows "X of Y Lessons Completed" in the sidebar. Compare Y against how many lessons you've found. If they don't match, you're missing pages.
  6. Keep paginating until you've found all lessons. A course claiming 16 lessons but only showing 10 means there's a page 2.
- Body content varies by theme:
  - Try `.post-body`, `.kjb-post__body`, `.post__body` first
  - If those are empty, check `.panel__block` inside `.section__body` — strip the `h1.panel__title` and `h5.panel__sub-title`, keep the remaining `h2`, `p`, and other content elements
  - Also check the course listing page for `p.syllabus__text` descriptions (these may be truncated — always prefer the full content from the lesson page)
  - **Do not skip body scraping** — always visit each individual lesson page and extract all text/HTML content. This is the lesson's copy and must be imported.

**WordPress/LearnDash:**
- Login at `/wp-login.php`
- Check for Vimeo, Wistia, YouTube iframes in lesson pages
- Body content in `.entry-content` or `.learndash-content`

**Teachable:**
- Login at `/sign_in`
- Course curriculum available at `/courses/{slug}/curriculum`
- Videos often Wistia or direct MP4

### Extracting Video URLs
Look for these in order on each lesson page:
1. `wistia_async_{ID}` classes on divs → Wistia
2. `<iframe src="...player.vimeo.com/video/{ID}...">` → Vimeo
3. `<iframe src="...youtube.com/embed/{ID}...">` → YouTube
4. `videoStream` in JSON data → BunnyCDN HLS
5. `<video>` or `<source>` elements → direct MP4/HLS
6. `<iframe src="...soundcloud.com/...">` or `soundcloud.com` links → SoundCloud audio
7. `<audio>` elements → direct audio files

### Extracting Body Content (CRITICAL — do not skip)
Every lesson's text/copy content MUST be scraped and imported. This is the lesson description, instructions, or written content that accompanies the video/audio.

- **Always visit each individual lesson page** and extract the body content. Do not rely solely on listing page excerpts (they are often truncated).
- Look for content in these selectors (in order): `.post-body`, `.kjb-post__body`, `.entry-content`, `.lesson-content`, `.panel__block` (Kajabi — strip title elements)
- Preserve meaningful HTML structure (headings, lists, links, bold/italic)
- Strip navigation, player UI, "Mark as Complete" buttons, "Next Lesson" links, platform chrome — only keep the actual lesson copy
- Remove `data-*` attributes from extracted HTML for cleanliness
- If the body is truly just the video embed with no text, set body to empty string — but verify by checking multiple selectors first

### Extracting the Main Asset (not always video)
Not every lesson's primary content is a video. The main asset could be an **image** (infographic, map, diagram), a **PDF** (workbook, guide, checklist), an **audio file** (podcast, meditation, voice note), or something else entirely. Look for whatever the lesson is built around:

- **Image lessons:** Check `.player__video img`, `.post-hero img`, or the main content area for large images. Use the CDN URL (e.g., `kajabi-storefronts-production.kajabi-cdn.com/...`) rather than `/courses/downloads/` URLs which may redirect to HTML.
- **PDF lessons:** Some lessons are just a downloadable PDF with no video — the PDF is the main content, not an attachment. Upload it as the lesson's `asset_id`.
- **Audio lessons:** Look for `<audio>` elements, SoundCloud embeds, or direct `.mp3`/`.m4a` links. These go in `asset_id` just like video.
- **No media at all:** Some lessons are text-only (instructions, welcome messages). Set `asset_id` to null and put everything in the body.

Upload whatever the primary content is as the lesson's `asset_id` — Simplero supports video, audio, images, and PDFs as lesson assets. If a lesson has both a video AND supplementary files (PDFs, images), the video goes in `asset_id` and the rest go as attachments.

### Extracting Attachments/Resources
Look for ALL downloadable files on each lesson page — PDFs, images, spreadsheets, docs, everything:

- Download links (PDFs, docs, spreadsheets)
- **"Downloads" sidebar** — Kajabi often has a separate downloads section with files (check for links containing `/courses/downloads/` or file extensions)
- Image files (.jpg, .png) that are lesson content (not thumbnails/navigation)
- Google Drive links — convert to direct download: `https://drive.google.com/uc?export=download&id={FILE_ID}`
- Dropbox links — append `?dl=1` for direct download
- Resource lists in JSON metadata (Skool `resources` field)
- Any `<a>` tags pointing to downloadable files

**For Kajabi download URLs:** The `/courses/downloads/{id}/filename` pattern may return an HTML page instead of the file. If the downloaded file is suspiciously small (<10 KB) or is HTML, fall back to the direct CDN URL from the `img` tag or use Playwright to follow the download redirect and capture the actual file URL.

## Step 4: Download Media

### Video Downloads (CRITICAL: must include audio)
Use `yt-dlp` for all video platforms — it handles auth, format selection, and audio merging correctly.

**Wistia:**
```bash
yt-dlp -f "hd_mp4-720p/best[height<=720]" -o output.mp4 "https://fast.wistia.net/embed/iframe/{WISTIA_ID}"
```
Wistia files are pre-muxed (video+audio in one file), so no merge needed.

**Vimeo:**
```bash
yt-dlp -f "bestvideo[ext=mp4][height<=720]+bestaudio" --merge-output-format mp4 -o output.mp4 "https://player.vimeo.com/video/{ID}?h={HASH}"
```
Include the `?h=` hash parameter if present in the iframe src — it's required for private videos.

**YouTube:**
```bash
yt-dlp -f "bestvideo[ext=mp4][height<=720]+bestaudio" --merge-output-format mp4 -o output.mp4 "https://www.youtube.com/watch?v={ID}"
```

**BunnyCDN HLS (Skool):**
```bash
ffmpeg -y -headers "Referer: https://www.skool.com/\r\n" -i "{HLS_URL}" -c copy output.mp4
```
Use 1200s timeout. These streams are already muxed.

**SoundCloud:**
```bash
yt-dlp -f "bestaudio" -o output.mp3 "{SOUNDCLOUD_URL}"
```

**Direct MP4/HLS:**
```bash
yt-dlp -f "best[height<=720]" -o output.mp4 "{URL}"
# Or for raw HLS:
ffmpeg -y -i "{URL}" -c copy output.mp4
```

### After each download, verify audio exists:
```bash
ffprobe -v quiet -show_streams -print_format json output.mp4
```
Check that there's a stream with `codec_type: "audio"`. If missing, re-download with a different format selector that includes audio.

### Attachment Downloads
- Google Drive: `curl -sL "https://drive.google.com/uc?export=download&id={FILE_ID}" -o file.pdf`
- For large Google Drive files, handle the confirmation page
- Direct URLs: `curl -sL "{URL}" -o file.ext`
- Determine file type from Content-Type header or file extension

## Step 5: Upload to Simplero

### Simplero API Reference

**Base URL:** `https://simplero.com/api/v1`
**Auth header:** `X-API-Key: {key}`

**Upload an asset (video, audio, PDF):**
```bash
curl -s -X POST -H "X-API-Key: {key}" \
  -F "file=@video.mp4;type=video/mp4" \
  https://simplero.com/api/v1/assets.json
```
Returns `{"id": 12345, ...}`. Use the ID as `asset_id` on a lesson (for video/audio) or to create an attachment.

Content types: `video/mp4`, `audio/mpeg`, `application/pdf`, `application/vnd.openxmlformats-officedocument.*`, etc.

**Create a module:**
```
POST /courses/{course_id}/course_modules.json
{"title": "Module Name", "publish_status": "published", "position": 1}
```

**Create a lesson:**
```
POST /course_modules/{module_id}/course_lessons.json
{
  "title": "Lesson Title",
  "body": "<p>HTML content here</p>",
  "asset_id": 12345,
  "position": 1,
  "publish_status": "published",
  "default_playback_speed": 1.0,
  "allow_multiple_completions": true,
  "remember_playback_position": false
}
```
The last three fields are optional — only set them for meditation/workout/repeat-listen content (see Playback Settings section below).

**Update a lesson:**
```
PUT /course_lessons/{lesson_id}.json
{"body": "...", "asset_id": 12345}
```
Note: PUT endpoint is flat (`/course_lessons/{id}.json`), NOT nested under modules.

**Add a file attachment to a lesson:**
```
POST /course_lessons/{lesson_id}/attachments.json
{"asset_id": 12345}
```
The asset must be uploaded first via `/assets.json`.

**Add a link attachment to a lesson:**
```
POST /course_lessons/{lesson_id}/attachments.json
{"title": "Resource Name", "link": "https://example.com/resource"}
```
Use this as fallback when file download isn't possible.

**List courses:**
```
GET /courses.json
```

**Get course with modules and lessons:**
```
GET /courses/{id}.json
```

## Step 6: Execute the Import

Write a Python script that:

1. **Loads or scrapes** source data (cache scraped data to a JSON file so re-runs don't re-scrape)
2. **Checks existing state** — fetch the Simplero module to see which lessons already exist (match by title)
3. **Tracks progress** — use a state file (`import_state.json`) mapping `"module:lesson"` keys to Simplero lesson IDs or `"error:..."` strings
4. **For each missing lesson:**
   a. Download video/audio → verify it has audio → upload to Simplero → get asset_id → delete local file
   b. Download any attachment files (PDFs from Google Drive, etc.) → upload to Simplero → get asset_ids
   c. Create the lesson with title, body HTML, asset_id, position, publish_status
   d. Add attachments via the attachments endpoint
   e. Save progress to state file
5. **Errors are retryable** — mark errors in state but allow re-running to retry them
6. **Delete local files** after each upload to save disk space
7. **Log everything** to a log file AND stdout

Run the script and monitor progress. For large imports (100+ lessons), run in background with `nohup` and check periodically.

## Important Rules

- **Import ALL lesson content.** Every piece of a lesson must be imported: video/audio (as asset), body copy/description (as body HTML), downloadable files (as attachments), and resource links (as link attachments). Do not skip any of these — visit each lesson page individually.
- **Always verify audio** after downloading video. If `ffprobe` shows no audio stream, try a different format or download method.
- **Prefer downloading files** over linking. Download Google Drive PDFs, Dropbox files, etc. as actual file attachments rather than adding them as link attachments.
- **Body content is separate from attachments.** The lesson body is HTML text content (descriptions, instructions, copy). Attachments are downloadable files. Don't put download links in the body — use proper attachments.
- **Don't put resource links in body HTML.** Use Simplero's attachment system instead (file or link type).
- **Position matters.** Set lesson positions to maintain the original course order.
- **Rate limit.** Add small delays between API calls if doing many in sequence.
- **Resume gracefully.** The state file lets you re-run after errors or interruptions without re-creating existing lessons.
- **Clean titles.** Strip numbering prefixes if already handled by position (e.g., "01: " prefix when it's already position 1). But keep them if they're meaningful (e.g., "Step # 1 - ...").

## Playback Settings for Meditations, Workouts, and Repeat-Listen Content

When the content is a **meditation, guided visualization, workout, breathwork session, hypnosis, affirmation track**, or similar content that people listen to repeatedly and should experience at normal speed:

Set these fields when creating the lesson:
```json
{
  "default_playback_speed": 1.0,
  "allow_multiple_completions": true,
  "remember_playback_position": false
}
```

- **`default_playback_speed: 1.0`** — prevents the player from defaulting to a faster speed. Meditations, workouts, and similar content should always start at 1x.
- **`allow_multiple_completions: true`** — lets users mark the lesson complete more than once, since these are designed to be repeated.
- **`remember_playback_position: false`** — the lesson starts from the beginning each time instead of resuming where they left off, since users want the full experience each listen.

**How to detect this type of content:** Look at the lesson title and description for keywords like: meditation, guided, visualization, breathwork, workout, exercise, affirmation, hypnosis, tapping, EFT, yoga, relaxation, sleep, journaling prompt (audio), prayer. Also consider the context — if the entire course is a meditation series or wellness program, apply these settings to all lessons.
