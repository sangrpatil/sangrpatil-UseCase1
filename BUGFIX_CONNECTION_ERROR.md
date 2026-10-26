# Bug Fix: Connection Error in Phase 4 (Retry Logic & Resilience)

## Problem
**Error**: `httpcore2.ConnectError: [Errno 11001] getaddrinfo failed` (DNS resolution failure)
**Location**: Phase 4 supervisor trying to connect to LLM API
**Root Cause**: No retry logic for transient network errors; single failed connection attempt causes entire phase to fail

## Solution
Added **exponential backoff retry logic** with **timeouts** to handle transient network failures gracefully.

## Changes Made

### 1. `agents/orchestrator.py` - Import time module (line 45)
Added `import time` to support retry delays.

### 2. `agents/orchestrator.py` - Enhanced `_make_llm()` function (lines 154-174)
Added timeout and retry configuration to all LLM providers:

**Before**:
```python
def _make_llm():
    if LLM_PROVIDER == "anthropic":
        from langchain_anthropic import ChatAnthropic
        return ChatAnthropic(api_key=ANTHROPIC_API_KEY, model=CLAUDE_MODEL)
    # ... other providers
```

**After**:
```python
def _make_llm():
    """Create LLM instance with resilient timeout and connection settings."""
    if LLM_PROVIDER == "anthropic":
        from langchain_anthropic import ChatAnthropic
        return ChatAnthropic(
            api_key=ANTHROPIC_API_KEY,
            model=CLAUDE_MODEL,
            timeout=60.0,           # 60-second timeout
            max_retries=3,          # Built-in retries
        )
    # ... other providers with timeout
```

**Key additions**:
- `timeout=60.0`: Prevents hanging connections
- `max_retries=3`: Built-in LLM client retry logic

### 3. `agents/orchestrator.py` - Enhanced `_run_supervisor()` function (lines 505-548)
Added **exponential backoff retry loop** around supervisor invocation:

**Before**:
```python
try:
    result = agent.invoke({"messages": [HumanMessage(content=phase_goal)]})
except Exception as e:
    trace.fail(str(e))
    raise
```

**After**:
```python
max_retries = 3
retry_delay = 2  # seconds
last_error = None

for attempt in range(1, max_retries + 1):
    try:
        llm = _make_llm()
        agent = create_agent(llm, tools, system_prompt=SUPERVISOR_PROMPT)
        result = agent.invoke({"messages": [HumanMessage(content=phase_goal)]})
        return result
    except (ConnectionError, TimeoutError, OSError) as e:
        last_error = e
        if attempt < max_retries:
            print(f"[ORCHESTRATOR] Attempt {attempt}/{max_retries} failed with transient error...")
            print(f"[ORCHESTRATOR] Retrying in {retry_delay} seconds...")
            time.sleep(retry_delay)
            retry_delay *= 2  # Exponential backoff: 2s, 4s, 8s
        else:
            print(f"[ORCHESTRATOR] All {max_retries} attempts failed.")
    except Exception as e:
        last_error = e
        trace.fail(str(e))
        raise

if last_error:
    error_msg = f"Phase {phase_name} supervisor failed after {max_retries} retries: {last_error}"
    trace.fail(error_msg)
    raise RuntimeError(error_msg) from last_error
```

**Key features**:
- Catches transient errors: `ConnectionError`, `TimeoutError`, `OSError`
- Exponential backoff: retries with increasing delays (2s → 4s → 8s)
- 3 total attempts before giving up
- Clear logging of retry attempts
- Non-transient errors (bugs in code) fail immediately

## Retry Behavior

| Attempt | Delay Before Retry | Status |
|---------|-------------------|--------|
| 1       | —                 | Initial attempt |
| 2       | 2 seconds         | If transient error on attempt 1 |
| 3       | 4 seconds         | If transient error on attempt 2 |
| 4       | Fail              | If transient error on attempt 3 |

## When Retries Help
✅ **Transient network issues** (temporary connectivity loss, ISP hiccup)
✅ **Temporary API unavailability** (brief service interruption)
✅ **DNS resolution timeouts** (slow DNS servers)
✅ **Rate limiting** (API returns 429 with retry-after)

## When Retries Won't Help
❌ **Invalid API key** — Will fail immediately on all retries
❌ **Wrong API endpoint** — Connection refused consistently
❌ **Code bugs** — Retries won't fix logic errors
❌ **Permanent network outage** — All retries will fail

## Troubleshooting Connection Errors

If you still see connection errors after 3 retries:

### 1. Verify API Key Configuration
```bash
# Check .env file exists and has ANTHROPIC_API_KEY set
cat .env | grep ANTHROPIC_API_KEY
```

### 2. Test Connectivity
```bash
# Test direct connection to Anthropic API
curl -I https://api.anthropic.com/
# Should return HTTP 200 or 401 (not connection error)
```

### 3. Check Python Environment
```bash
# Verify required packages are installed
pip list | grep -E "langchain|anthropic|groq"
```

### 4. View Detailed Logs
Look at `audit_logs/` for the run_id that failed:
```bash
# Find the most recent audit log
ls -ltr audit_logs/ | tail -1
```

### 5. Try Manual LLM Test
```python
from core.config import LLM_PROVIDER, ANTHROPIC_API_KEY
from langchain_anthropic import ChatAnthropic

llm = ChatAnthropic(api_key=ANTHROPIC_API_KEY, model="claude-haiku-4-5-20251001")
response = llm.invoke("Say hello")
print(response)
```

## Impact
- ✅ Phase 4 tolerates temporary network hiccups
- ✅ Better user experience with automatic retries
- ✅ Clearer logging of connection attempts
- ✅ Configurable retry behavior (can be adjusted if needed)
- ✅ Non-transient errors still fail fast and clearly

## Testing
To verify the fix:
1. Run orchestrator with Phase 4
2. If you see transient errors, they will retry automatically
3. Check console logs for `[ORCHESTRATOR] Attempt N/3` messages
4. Phase should eventually succeed or fail with clear error message

## Related Configuration Files
- `.env` — Contains `ANTHROPIC_API_KEY`, `GROQ_API_KEY`, `GOOGLE_API_KEY`
- `core/config.py` — LLM provider selection and model configuration
- `agents/orchestrator.py` — Supervisor agent orchestration
