# ALFie (alf-gchat-bot) — Portfolio Showcase

## Context

ALFie is a **production internal tool** — an AI-powered staff assistant deployed inside Google Chat for Alabama Forward, a nonpartisan civic engagement nonprofit. It is not a demo or prototype; it serves ~15 staff members daily at a cost of ~$8/month.

The architecture reflects that context: a serverless, two-function design on Google Cloud that prioritizes **reliability over sophistication** and **graceful degradation over feature richness**. Every design choice trades off against a real constraint — Cloud Function timeouts, Google Chat API rate limits, Firestore pricing tiers, and a small team that cannot afford on-call pages. The codebase is intentionally readable and modular so that a single developer (the author) can maintain, debug, and extend it without tribal knowledge.

---

## Skills Demonstrated

| Skill | Where It Appears |
|---|---|
| Python (3.13) | All application code (`app/main.py`, `app/context.py`, `app/streaming.py`, etc.) |
| Google Cloud Functions (Gen 2) | Two-function serverless architecture (`main.py:chat_webhook`, `async_worker.py:process_async_response`) |
| Google Cloud Firestore | Conversation history persistence with TTL expiration (`conversation_store.py`) |
| Google Cloud Tasks | Async job queuing with OIDC authentication (`async_worker.py:create_async_task`) |
| REST API Integration | Direct Anthropic Claude API calls with SSE streaming (`streaming.py`, `main.py:query_claude`) |
| CI/CD with GitHub Actions | 7-stage pipeline: lint, security scan, unit tests, integration tests, coverage, deploy, auto-rollback (`.github/workflows/ci-cd.yml`) |
| Rate Limiting & Backoff | Sliding-window rate limiter with adaptive delays and exponential backoff on 429s (`rate_limiter.py`, `chat_api.py:update_message_with_retry`) |
| Prompt Engineering | Dynamic system prompt construction with context injection, query-type detection, and parameter tuning (`context.py:ContextManager`) |
| Test-Driven Development | Unit and integration test suites with pytest, mocking external APIs, >40% coverage enforcement (`tests/unit/`, `tests/integration/`) |

---

## Architectural Decisions

### 1. Two-Function Split: Webhook + Async Worker

**Problem:** Google Chat requires a synchronous HTTP response within seconds, but Claude API calls can take 30-60 seconds for complex queries. A single Cloud Function would timeout.

**Approach chosen:** Split into two Cloud Functions — a lightweight webhook that returns immediately, and a heavy worker triggered via Cloud Tasks. Short messages (greetings, <15 chars) are handled synchronously; everything else is queued.

**Why not a single function with background threads?** Cloud Functions Gen 2 can handle background work, but the 5-minute timeout still applies to the overall invocation. The two-function design gives the worker its own 9-minute timeout and decouples user-facing latency from processing time.

```python
# app/main.py:44-66
def should_process_async(query_type: str, message_length: int) -> bool:
    """
    Strategy: Process ALL queries async except very short ones (greetings).
    This ensures users always get complete responses without timeouts.
    """
    if message_length > 15:
        logger.info("Query (%d chars) flagged for async processing", message_length)
        return True

    logger.info("Short query (%d chars) - processing synchronously", message_length)
    return False
```

### 2. Firestore for Conversation History (Not the Chat API)

**Problem:** The Google Chat API does not support reading DM history, and its Space history API requires Developer Preview enrollment. The bot needs conversation context for both DMs and Spaces.

**Approach chosen:** Store all messages in Firestore with a differentiated strategy — DMs get deep context (50 stored, 15 retrieved) and Spaces get shallow context (15 stored, 5 retrieved) to respect group privacy. A 7-day TTL auto-deletes stale data.

**Why not an in-memory cache?** Cloud Functions are stateless — each invocation may run on a different instance. Firestore provides persistence across invocations with no cold-start penalty, and native TTL support handles data retention without a cron job.

```python
# app/config.py:258-290 (abridged)
CONVERSATION_HISTORY_CONFIG = {
    # DM: Deep context for project continuity
    "dm_max_history": 50,       # Store ~1 week of active conversation
    "dm_max_messages": 15,      # Retrieve last 15 messages for Claude
    # SPACES: Shallow context for privacy and relevance
    "space_max_history": 15,    # Store only recent messages
    "space_max_messages": 5,    # Retrieve last 5 messages for Claude
    "space_thread_only": True,  # Only fetch from current thread
    # TTL
    "ttl_days": 7,              # Auto-expire after 7 days of inactivity
}
```

### 3. Feature-Flagged Streaming with Automatic Batch Fallback

**Problem:** Users experienced long waits (30-60s) before seeing any response. Streaming gives real-time feedback, but it introduces complexity — rate limits, partial failures, and a fundamentally different code path.

**Approach chosen:** Streaming is gated behind a config toggle (`STREAMING_CONFIG["enabled"]`). When enabled, the worker posts a placeholder message and PATCHes it as Claude generates text. If streaming fails at any point, the system automatically falls back to the original batch mode. No user action required.

**Why not just always stream?** The batch path is battle-tested and simpler to debug. Keeping it as an automatic fallback means a streaming regression never silences the bot — it just temporarily reverts to the old behavior.

```python
# app/async_worker.py:326-348 (abridged)
use_streaming = STREAMING_CONFIG.get("enabled", False)

if use_streaming:
    try:
        claude_response = process_streaming_response(
            prompt=prompt, custom_parameters=optimal_params,
            space_name=space_name, thread_name=thread_name,
            chat_client=chat_client,
        )
    except Exception as stream_err:
        logger.error("Streaming failed, falling back to batch: %s", str(stream_err))
        use_streaming = False
        claude_response = query_claude_sync(prompt, optimal_params)
else:
    claude_response = query_claude_sync(prompt, optimal_params)
```

---

## Code Snippets (Core Competencies)

### 1. Server-Sent Events (SSE) Streaming Parser

**Skills:** REST API integration, Python generators, streaming I/O

This generator function sends a request to the Claude Messages API with `stream: True`, then parses the raw SSE byte stream line-by-line, extracting only `content_block_delta` events that carry generated text. Using `yield` makes it composable — the caller controls the consumption rate without buffering the entire response.

```python
# app/streaming.py:38-119
def stream_claude_response(
    prompt: str,
    custom_parameters: Optional[Dict] = None,
) -> Iterator[str]:
    """Stream text deltas from the Claude API via Server-Sent Events."""
    parameters = custom_parameters or MODEL_PARAMETERS

    request_body = {
        "model": CLAUDE_MODEL,
        "messages": [{"role": "user", "content": prompt}],
        "max_tokens": parameters.get("max_output_tokens"),
        "temperature": parameters.get("temperature"),
        "stream": True,
    }

    response = requests.post(
        "https://api.anthropic.com/v1/messages",
        json=request_body, headers=headers,
        stream=True,   # Don't download full body upfront
        timeout=240,
    )
    response.raise_for_status()

    for line in response.iter_lines():
        if not line:
            continue
        line = line.decode("utf-8")
        if not line.startswith("data: "):
            continue

        data = json.loads(line[6:])  # Strip "data: " prefix

        if data.get("type") == "content_block_delta":
            delta = data.get("delta", {})
            if delta.get("type") == "text_delta":
                text = delta.get("text", "")
                if text:
                    yield text

        if data.get("type") == "message_stop":
            break
```

### 2. Sliding-Window Rate Limiter

**Skills:** Rate limiting, algorithm design, defensive programming

The Google Chat API enforces 60 writes/minute per space. This class implements a sliding-window counter that tracks per-space write timestamps and calculates adaptive delays. Instead of a binary allow/deny, the delay increases gradually (0.5s at 11 writes, up to 3.0s at 41+ writes), which smooths throughput rather than creating bursts-and-pauses.

```python
# app/rate_limiter.py:34-131
class RateLimiter:
    def __init__(self, max_per_minute: int = 50):
        self.max_per_minute = max_per_minute
        self.space_timestamps: Dict[str, List[float]] = defaultdict(list)

    def _prune_window(self, space_name: str) -> None:
        """Remove timestamps older than 60 seconds."""
        cutoff = time.time() - 60
        self.space_timestamps[space_name] = [
            ts for ts in self.space_timestamps[space_name] if ts > cutoff
        ]

    def wait_if_needed(self, space_name: str) -> float:
        self._prune_window(space_name)
        recent_count = len(self.space_timestamps[space_name])

        # Hard limit: wait for oldest write to exit the window
        if recent_count >= self.max_per_minute:
            oldest = self.space_timestamps[space_name][0]
            wait_time = (oldest + 60) - time.time() + 0.1
            if wait_time > 0:
                time.sleep(wait_time)
                return wait_time

        # Adaptive delay: gradual slowdown approaching the limit
        if recent_count <= 10: return 0.0
        if recent_count > 40: delay = 3.0
        elif recent_count > 30: delay = 2.0
        elif recent_count > 20: delay = 1.0
        else: delay = 0.5

        time.sleep(delay)
        return delay

    def record_write(self, space_name: str) -> None:
        self.space_timestamps[space_name].append(time.time())
```

### 3. Token-Aware Conversation Context Trimming

**Skills:** Context window management, data structures, LLM prompt optimization

Claude has a finite context window, and stuffing too much history into the prompt degrades response quality. This method trims conversation history to fit a token budget while always preserving the two most recent messages (which carry the strongest conversational signal). It works backward from the end, adding older messages until the budget is exhausted.

```python
# app/context.py:236-293
def trim_conversation_context(
    self, messages: List[Dict[str, Any]], max_tokens: int = 2000
) -> List[Dict[str, Any]]:
    """Trim oldest messages if context exceeds token limit.
    Always keep most recent 2 messages."""
    if not messages or len(messages) <= 2:
        return messages

    total_tokens = sum(
        self.estimate_token_count(msg.get("text", "")) for msg in messages
    )
    if total_tokens <= max_tokens:
        return messages

    # Start with last 2 messages (always kept)
    trimmed_messages = messages[-2:]
    trimmed_tokens = sum(
        self.estimate_token_count(msg.get("text", "")) for msg in trimmed_messages
    )

    # Add older messages from the end until budget is exhausted
    for i in range(len(messages) - 3, -1, -1):
        msg_tokens = self.estimate_token_count(messages[i].get("text", ""))
        if trimmed_tokens + msg_tokens <= max_tokens:
            trimmed_messages.insert(0, messages[i])
            trimmed_tokens += msg_tokens
        else:
            break

    return trimmed_messages
```

### 4. Dynamic Query-Type Detection and Parameter Tuning

**Skills:** NLP heuristics, strategy pattern, configuration-driven design

Instead of sending every query to Claude with identical parameters, this system classifies each message by topic and adjusts temperature and token limits accordingly. Legal queries get low temperature (0.1) for consistency; arts/culture queries get higher temperature (0.5) for creativity. The detection is keyword-based with explicit priority ordering to prevent ambiguous matches.

```python
# app/context.py:59-91
def detect_query_type(self, user_message: str) -> str:
    """Detect the type of query to select appropriate parameters."""
    message_lower = user_message.lower()

    # Priority order: Check more specific triggers first
    priority_order = [
        "coding_analysis",      # Most specific technical terms
        "grant_writing",        # Specific funding terms
        "legal_official",       # Specific legal/registration terms
        "community_events",     # Event-specific terms
        "community_outreach",   # Outreach-specific terms
        "arts_culture",         # Arts/culture terms
        "quick_greeting",       # Most generic — checked last
    ]

    for param_type in priority_order:
        keywords = self.parameter_triggers.get(param_type, [])
        if any(keyword in message_lower for keyword in keywords):
            return param_type

    return "standard"
```

### 5. Markdown-to-Google Chat Format Converter

**Skills:** Regex, text processing, platform-specific API adaptation

Google Chat uses a non-standard markdown dialect (`*bold*` instead of `**bold**`, no table support). This function translates standard markdown — tables get wrapped in code blocks, headings become bold, nested bold within headings is collapsed, and bullet markers are replaced with Unicode bullets. The table detection uses a line-by-line state machine.

```python
# app/utils.py:229-302
def convert_markdown_to_gchat(text: str) -> str:
    # Convert markdown tables to code blocks (Google Chat doesn't support tables)
    lines = text.split("\n")
    in_table = False
    converted_lines = []

    for i, line in enumerate(lines):
        stripped = line.strip()
        if stripped.startswith("|") or (stripped.count("|") >= 2):
            if not in_table:
                converted_lines.append("```")
                in_table = True
            converted_lines.append(line)
        else:
            if in_table:
                converted_lines.append("```")
                in_table = False
            converted_lines.append(line)

    if in_table:
        converted_lines.append("```")

    text = "\n".join(converted_lines)

    # Headings with nested bold: ### **Text** -> *Text*
    text = re.sub(r"^#{1,6}\s+\*\*(.+?)\*\*", r"*\1*", text, flags=re.MULTILINE)

    # Plain headings: ### Text -> *Text*
    def replace_header(match):
        content = match.group(1).replace("**", "")
        return f"*{content}*"
    text = re.sub(r"^#{1,6}\s+(.+?)$", replace_header, text, flags=re.MULTILINE)

    # Bold: **text** -> *text*
    text = re.sub(r"\*\*(.+?)\*\*", r"*\1*", text)
    # Italic: __text__ -> _text_
    text = re.sub(r"__(.+?)__", r"_\1_", text)
    # Bullets: - item or * item -> bullet
    text = re.sub(r"^\s*[-*]\s+", "  \u2022 ", text, flags=re.MULTILINE)

    return text
```

---

## Clever Implementations

### 1. StreamAccumulator: Dual-Threshold Buffering for Rate-Limited APIs

The naive approach to streaming is to patch the message on every delta (too fast — hits rate limits) or on a fixed timer (too slow — wastes time when text arrives in bursts). The `StreamAccumulator` solves this with a dual-threshold trigger: it fires when **either** 150 new characters have accumulated **or** 1 second has passed since the last update (with a minimum of 50 chars to avoid trivial patches). This means fast-generating text gets batched into meaningful visual updates, while slow-generating text still appears promptly.

The design is also shaped by Google Chat's PATCH semantics — the API *replaces* the entire message text, so the accumulator must return the **full** accumulated text on each trigger, not just the delta. This is counterintuitive if you're used to append-based streaming.

```python
# app/streaming.py:122-212
class StreamAccumulator:
    """
    Buffers streaming text deltas and signals when to patch the Chat message.
    Triggers an update when either:
      - Enough new characters have accumulated (char_threshold), OR
      - Enough time has passed since the last update (update_interval)
    Returns the FULL accumulated text (not just the new part), because
    Google Chat's PATCH endpoint replaces the entire message text.
    """
    def __init__(self, update_interval=1.0, char_threshold=150, min_update_size=50):
        self.full_text = ""
        self.last_update_text = ""
        self.last_update_time = time.time()
        self.update_interval = update_interval
        self.char_threshold = char_threshold
        self.min_update_size = min_update_size

    def add_text(self, text_delta: str) -> Optional[str]:
        """Add a delta. Returns full text if an update should be sent, else None."""
        self.full_text += text_delta

        new_chars_count = len(self.full_text) - len(self.last_update_text)
        time_since_update = time.time() - self.last_update_time

        should_update = (new_chars_count >= self.char_threshold) or (
            time_since_update >= self.update_interval
            and new_chars_count >= self.min_update_size
        )

        if should_update:
            self.last_update_text = self.full_text
            self.last_update_time = time.time()
            return self.full_text

        return None

    def force_update(self) -> str:
        """Force-flush after stream ends to ensure final text is sent."""
        self.last_update_text = self.full_text
        self.last_update_time = time.time()
        return self.full_text

    def has_pending_changes(self) -> bool:
        return self.full_text != self.last_update_text
```

### 2. Dual-Storage Fallback for Space Conversations

Google Chat's threading model is inconsistent — sometimes a `thread_id` is present, sometimes it is not, and the same conversation can appear under different thread identifiers. Instead of trying to normalize this, the bot stores every Space message **twice**: once under the thread-specific key and once under the space-level key. On retrieval, it tries thread-specific first and falls back to space-level if no messages are found. This makes context retrieval resilient to Google Chat's threading quirks without requiring complex ID reconciliation.

```python
# app/main.py:319-374 (saving side)
# For DMs: Use thread_id=None to store all messages together
# For Spaces: Save to BOTH thread-specific AND space-level for fallback
if space_type == "DM":
    save_thread_id = None
else:
    save_thread_id = thread_name

# Save to thread-specific location
conv_store.save_message(space_id=space_name, thread_id=save_thread_id, ...)

# For Spaces: ALSO save to space-level for fallback
if space_type == "ROOM":
    conv_store.save_message(space_id=space_name, thread_id=None, ...)

# app/main.py:251-275 (retrieval side)
if space_type == "ROOM":
    # Try thread-specific first
    recent_messages = conv_store.get_conversation(
        space_id=space_name, thread_id=thread_name, max_messages=max_msgs
    )
    # If no messages found in thread, fall back to entire space
    if len(recent_messages) == 0 and thread_name is not None:
        recent_messages = conv_store.get_conversation(
            space_id=space_name, thread_id=None, max_messages=max_msgs
        )
```
