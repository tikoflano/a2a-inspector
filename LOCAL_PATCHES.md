# A2A Inspector — Local Patches

Issues encountered while testing with Gradial's A2A endpoint, and the local code changes made to fix them. These may become upstream PRs.

## 1. Browser headers leaked to upstream agent (causes 500s)

**Problem:** The backend extracts custom headers from the incoming browser request to forward to the agent. But the `STANDARD_HEADERS` denylist is too narrow — browser headers like `sec-ch-ua`, `sec-fetch-*`, `origin`, `referer`, `cookie` pass through and get forwarded. Gradial returns HTTP 500 when it receives these unexpected headers.

**Related issue:** [#101 — Custom headers casing is changed?](https://github.com/a2aproject/a2a-inspector/issues/101) (similar area, different symptom)

**Fix (`backend/app.py`):** Expanded `STANDARD_HEADERS` set to include browser-specific headers:

```python
STANDARD_HEADERS = {
    'host',
    'user-agent',
    'accept',
    'content-type',
    'content-length',
    'connection',
    'accept-encoding',
    'accept-language',
    'origin',
    'referer',
    'cookie',
    'sec-ch-ua',
    'sec-ch-ua-mobile',
    'sec-ch-ua-platform',
    'sec-fetch-site',
    'sec-fetch-mode',
    'sec-fetch-dest',
}
```

## 2. Blocking `message/send` sent to non-streaming agents

**Problem:** The inspector creates the A2A client with `polling=False` (the default), which makes the SDK send `blocking: true` in the `message/send` configuration. Agents that don't support streaming or blocking (like Gradial) reject with error `-32004: "Blocking message/send is not supported"`.

**Fix (`backend/app.py`):** Set `polling=True` in `ClientConfig` when the agent card declares `streaming: false`:

```python
supports_streaming = (
    card.capabilities.streaming if card.capabilities else False
)
a2a_config = ClientConfig(
    ...,
    polling=not supports_streaming,
)
```

## 3. Metadata values sent as strings instead of objects

**Problem:** The frontend `getMessageMetadata()` uses `getKeyValuePairs()` which returns `Record<string, string>`. When metadata values are JSON objects (e.g., Gradial requires `metadata.gradial` to be `{workspaceId: ..., environmentId: ...}`), they get sent as raw strings. Gradial rejects with `-32602: "Expected object, received string"`.

**Fix (`frontend/src/script.ts`):** Changed `getMessageMetadata()` to attempt `JSON.parse()` on each value:

```typescript
function getMessageMetadata(): Record<string, unknown> {
  const raw = getKeyValuePairs(
    metadataList,
    '.metadata-item',
    '.metadata-key',
    '.metadata-value',
  );
  const parsed: Record<string, unknown> = {};
  for (const [key, value] of Object.entries(raw)) {
    try {
      parsed[key] = JSON.parse(value);
    } catch {
      parsed[key] = value;
    }
  }
  return parsed;
}
```

After changing, rebuild with: `cd frontend && npm run build`
