# Plan: Lecture Transcript Context API (PoC)

## Goal

FastAPI endpoint that accepts a video timestamp and returns the audio transcript up to that point, formatted for LLM context injection.

## Design

**Audio-only.** The video OCR data is too noisy for a PoC. The professor's speech already covers slide content. We can revisit video transcript later if needed.

**Response shape:**

```json
{
  "lecture_id": "lec10",
  "current_time": 300.0,
  "transcript": "All right, welcome data 100. So today we are going to start modeling..."
}
```

`transcript` is a plain text string — concatenated speech segments from `t=0` to `t=current_time`. The Open WebUI filter can inject it directly into a system prompt.

**Endpoint:**

```
POST /context
Body: { "lecture_id": "lec10", "current_time": 300.0 }
```

`lecture_id` defaults to `"lec10"` (only lecture we have). `current_time` is seconds into the video. For the PoC, this is passed manually via curl. Later, the frontend will send it automatically when the user chats.

## Implementation

### Step 1: Transcript loading

**File:** `askademia-api/askademia/transcripts.py` (new)

- Load `data/audio_transcript_lec10.json` into a list of dicts at startup
- Sort by timestamp (should already be sorted, but be safe)
- Store in a module-level dict keyed by lecture ID
- `get_transcript_up_to(lecture_id, current_time) -> str` — filters segments where `timestamp <= current_time`, joins their `text` fields with spaces

### Step 2: API endpoint

**File:** `askademia-api/askademia/main.py` (modify)

- Import transcript loading, call it in a lifespan handler
- Add `POST /context` with a Pydantic request model
- Return `{"lecture_id", "current_time", "transcript"}`
- 404 if lecture_id not found

## Files

| File | Action |
|------|--------|
| `askademia-api/askademia/transcripts.py` | Create |
| `askademia-api/askademia/main.py` | Modify |

## Test

```bash
curl -X POST http://localhost:8000/context \
  -H "Content-Type: application/json" \
  -d '{"current_time": 300}'
```

## Future Work

- Video OCR transcript merging (clean + interleave with audio)
- Open WebUI filter that calls `/context` and injects into system prompt
- VideoPlayer exposing `currentTime` to chat flow
- Multiple lecture support
- Token budget / context window management
