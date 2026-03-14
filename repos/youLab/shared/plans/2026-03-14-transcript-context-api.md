# Plan: Lecture Transcript Context API

## Goal

Build a FastAPI endpoint in askademia-api that accepts a video timestamp and returns merged audio + video transcript context from the start of the lecture up to that point, formatted for LLM consumption.

## Key Design Decisions

### 1. Transcript Merge Strategy: Interleaved Timeline

Rather than keeping audio and video transcripts separate, **interleave them chronologically** into a single timeline. This gives the LLM a natural reading order that mirrors what the student experienced.

**Why interleave?**
- The LLM sees the same flow a student does: slide appears, then professor explains it
- Avoids the LLM needing to mentally correlate two separate lists by timestamp
- Naturally handles the different granularity (208 video segments vs 534 audio segments)

**Merge algorithm:**
1. Filter both lists to segments where `timestamp <= current_time`
2. Tag each segment with its source type (`audio` or `slide`)
3. Sort all segments by timestamp
4. For video segments: a slide is "active" from its timestamp until the next video segment's timestamp. We only include the slide text once (when it first appears), not repeated for every audio segment during that slide.

### 2. Video OCR Text Cleaning

The raw video OCR text contains significant noise — browser chrome, URLs, tab titles, timestamps, UI elements like "X", "K", "+", etc. This needs to be cleaned before sending to the LLM.

**Cleaning strategy:**
- Strip common browser UI patterns (tab bars, URL bars, navigation elements)
- Remove OCR artifacts (single characters like "X", "K", "O", "H", etc.)
- Remove timestamps in the format `2024-10-01 11:10:50`
- Remove Google Slides URLs
- Remove professor name / metadata that repeats on every slide
- Keep the actual slide content (titles, bullet points, equations, code)

**Implementation:** A `clean_ocr_text(raw_text: str) -> str` function that applies regex/string filters. This is isolated so we can tune it without touching the merge logic.

### 3. Significant Slide Detection via Diff Scores

Not every video segment represents a meaningful slide change. Many are minor variations (mouse movement, highlighting, cursor changes). Use the `bert_diff` score to filter out noise.

**Threshold strategy:**
- Only include video segments where `bert_diff >= 0.10` (meaningful semantic change in slide content)
- This filters ~208 segments down to the significant slide transitions
- The threshold is configurable for tuning

**Why `bert_diff` over `diff`?**
- `diff` measures pixel-level changes (catches mouse moves, cursor blinks)
- `bert_diff` measures semantic text change (catches actual content changes)
- A slide with new content highlighted has high `bert_diff` but possibly low `diff`

### 4. Output Format: Structured Markdown

Format the merged transcript as **markdown sections** for LLM context injection. This is readable by the LLM and clearly delineates what's spoken vs what's on screen.

```
## Lecture Transcript (0:00 - 12:34)

[SLIDE at 0:07]
LECTURE 10
Introduction to Modeling, SLR
Understanding the usefulness of models and the simple linear regression model

[AUDIO at 0:03]
All right, welcome data 100. So today we are going to start modeling...

[AUDIO at 0:10]
So just before I start, I want to remind you...

[SLIDE at 0:30]
Plan for Next Few Lectures: Modeling
- Modeling I: Intro to Modeling, Simple Linear Regression
- Modeling II: Different models, loss functions, linearization
- Modeling III: Multiple Linear Regression

[AUDIO at 0:25]
So kind of telling you the big picture of what's going to happen...
```

**Why markdown?**
- LLMs parse markdown naturally
- Clear visual separation between slide content and spoken content
- Timestamps provide temporal grounding
- No complex JSON structure to parse mid-prompt

### 5. API Response Shape

```json
{
  "lecture_id": "lec10",
  "current_time": 754.5,
  "formatted_context": "## Lecture Transcript (0:00 - 12:34)\n\n[SLIDE at 0:07]...",
  "segment_count": {
    "audio": 87,
    "slides": 12
  }
}
```

**Why include `formatted_context` as a pre-rendered string?**
- The Open WebUI filter can inject it directly into the system prompt without additional processing
- No client-side formatting logic needed
- The API owns the formatting, making it easy to iterate on prompt quality

### 6. Frontend Timestamp Communication

The frontend will send the current video timestamp as a query parameter or POST body field when the user sends a chat message. Two integration points:

**Phase 1 (this plan):** The API endpoint accepts timestamp directly — testable via curl/Postman.

**Phase 2 (future):** The VideoPlayer component exposes `currentTime` to the chat flow:
- VideoPlayer emits a `timeupdate` event or exposes a bindable `currentTime` prop
- Chat.svelte reads the current timestamp when assembling the message payload
- The Open WebUI filter calls the askademia-api `/context` endpoint with the timestamp
- Filter injects the returned `formatted_context` into the system prompt

## Implementation Steps

### Step 1: Data Models and Loading

**File:** `askademia-api/askademia/transcripts.py` (new)

- Define `AudioSegment` and `VideoSegment` Pydantic models matching the JSON schemas
- Define `LectureTranscripts` container that holds both lists for a lecture
- `load_transcripts(lecture_id: str) -> LectureTranscripts` function that reads from `data/` directory
- Load at module level (or via FastAPI lifespan) so data is in memory at startup
- Path resolution: use `Path(__file__).parent.parent.parent / "data"` to find the data directory relative to the package

**Audio segment model:**
```python
class AudioSegment(BaseModel):
    timestamp: float
    text: str
    start_ms: int
    end_ms: int
    duration_ms: int
    word_count: int
    avg_confidence: int
```

**Video segment model:**
```python
class VideoSegment(BaseModel):
    timestamp: float
    text: str
    diff: float
    bert_diff: float
    bert_2_diff: float
    bert_3_diff: float
```

### Step 2: OCR Text Cleaning

**File:** `askademia-api/askademia/transcripts.py` (same file)

`clean_ocr_text(raw_text: str) -> str` function:

1. Split text by newlines
2. Remove lines that match known noise patterns:
   - Lines that are just 1-2 characters (OCR artifacts: "X", "K", "O", "H", "+", etc.)
   - Lines matching `\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}` (recording timestamps)
   - Lines containing `docs.google.com` (slide URLs)
   - Lines starting with common browser chrome patterns (`lec10.ipynb`, `Lec 10 - DS100`)
   - Lines that are just "Narges Norouzi" (repeated professor name)
   - Lines matching common UI noise: `-->`, `...`, `ab`, `ab/`, numbers that look like IDs (e.g., "1380664")
3. Strip leading/trailing whitespace from remaining lines
4. Collapse multiple blank lines into one
5. Return joined text

### Step 3: Transcript Merge and Formatting

**File:** `askademia-api/askademia/context.py` (new)

`build_context(transcripts: LectureTranscripts, current_time: float, bert_diff_threshold: float = 0.10) -> ContextResponse` function:

1. Filter audio segments: `[s for s in transcripts.audio if s.timestamp <= current_time]`
2. Filter video segments: `[s for s in transcripts.video if s.timestamp <= current_time and s.bert_diff >= bert_diff_threshold]`
3. Clean each video segment's text with `clean_ocr_text()`
4. Skip video segments where cleaned text is empty or very short (< 10 chars)
5. Create tagged entries: `(timestamp, "audio"|"slide", text)`
6. Sort all entries by timestamp
7. Format as markdown:
   - Header: `## Lecture Transcript (0:00 - {format_time(current_time)})`
   - Each entry: `[SLIDE at {format_time(ts)}]\n{text}\n` or `[AUDIO at {format_time(ts)}]\n{text}\n`
   - `format_time(seconds)` -> `"M:SS"` or `"H:MM:SS"`
8. Return `ContextResponse` with formatted string and counts

### Step 4: API Endpoint

**File:** `askademia-api/askademia/main.py` (modify existing)

Add `POST /context` endpoint:

```python
class ContextRequest(BaseModel):
    lecture_id: str = "lec10"
    current_time: float  # seconds into the video

class ContextResponse(BaseModel):
    lecture_id: str
    current_time: float
    formatted_context: str
    segment_count: dict  # {"audio": int, "slides": int}

@app.post("/context", response_model=ContextResponse)
async def get_context(req: ContextRequest):
    transcripts = get_transcripts(req.lecture_id)
    if not transcripts:
        raise HTTPException(404, f"Lecture '{req.lecture_id}' not found")
    return build_context(transcripts, req.current_time)
```

Also add transcript loading in a lifespan handler:

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    load_all_transcripts()
    yield

app = FastAPI(title="Askademia API", version="0.1.0", lifespan=lifespan)
```

### Step 5: Add `httpx` dependency (already present) and test manually

No new dependencies needed — fastapi, pydantic, and the stdlib are sufficient.

**Manual test:**
```bash
# Start the API
make dev-api

# Get context at 5 minutes into the lecture
curl -X POST http://localhost:8000/context \
  -H "Content-Type: application/json" \
  -d '{"lecture_id": "lec10", "current_time": 300}'
```

## File Summary

| File | Action | Description |
|------|--------|-------------|
| `askademia-api/askademia/transcripts.py` | Create | Data models, loading, OCR cleaning |
| `askademia-api/askademia/context.py` | Create | Merge logic, formatting, response building |
| `askademia-api/askademia/main.py` | Modify | Add `/context` endpoint, lifespan loading |

## Out of Scope (Future Work)

- **Open WebUI filter/pipe** that calls `/context` and injects into system prompt
- **VideoPlayer timestamp binding** — exposing `currentTime` from the Mux player to Svelte state
- **Frontend integration** — passing timestamp through the chat message flow
- **Multiple lectures** — for now, only lecture 10 data is loaded
- **Streaming/chunked context** — for very long lectures, may need to window or summarize older segments
- **Token budget management** — truncating context if it exceeds LLM context window limits

## Open Questions

1. **bert_diff threshold tuning** — 0.10 is an educated guess. May need adjustment after seeing real output. Consider logging filtered-out segments to tune.
2. **Deduplication** — Some consecutive video segments may have nearly identical cleaned text (same slide, minor OCR variation). Should we deduplicate consecutive identical slides? (Probably yes — compare cleaned text and skip if same as previous.)
3. **First slide exception** — The first video segment always has `bert_diff` relative to nothing, so its score may not be meaningful. Should always include the first segment regardless of threshold.
