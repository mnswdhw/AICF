# AI Changefeed (AICF) — v0.1 **Minimal Core** (Community Draft)

**Status:** Draft <br>
**Authors:** Manas Wadhwa (manaswadhwa41@gmail.com) <br>
**License:** CC BY 4.0  
**Goal:** A tiny, web-native way for publishers to expose **append-only change events** so consumers (crawlers, agents, RAG systems) refresh **only what changed**.  
**Non-Goals:** Replace sitemaps/RSS, dictate crawler policy, or standardize a chunking algorithm.

> **Keep it simple:** v0.1 does **section-level** invalidation with **anchors**.  
> Chunks, precomputed vectors, signatures, push delivery, and billing are intentionally **out of scope** for the minimal core.

---

## 0. RFC Keywords
“MUST”, “SHOULD”, and “MAY” are as defined in RFC 2119.

---

## 1. Discovery

Publishers **MUST** expose:

```
GET /.well-known/ai-changefeed
Content-Type: application/json
```


**Fields**
- `aicf_version` (string, **REQUIRED**) — e.g., `"0.1"`.
- `self` (string, **REQUIRED**) — absolute URL of the NDJSON feed.
- `license` (string, OPTIONAL) — default license for events (e.g., `"CC BY 4.0"`).
- `about` (string, OPTIONAL) — human-readable page describing scope/terms.
- `ttl_seconds` (integer, OPTIONAL) — suggested cache time for the feed URL.

**Example**
```json
{
  "aicf_version": "0.1",
  "self": "https://publisher.example/ai-changes.ndjson",
  "license": "CC BY 4.0",
  "about": "https://publisher.example/ai-changefeed",
  "ttl_seconds": 60
}
```

## 2 Feed Format (NDJSON)

`self` MUST return append-only NDJSON (`application/x-ndjson`), oldest → newest. One JSON object per line = one event.

### 2.1 Required fields

* `id` (string) — unique, sortable per feed (ULID or timestamp+counter recommended).
* `action` (string) — `"create" | "update" | "delete"`.
* `url` (string) — absolute URL of the changed resource; MAY include a fragment (e.g., `#pricing`).
* `time` (string) — RFC 3339 UTC timestamp, e.g., `"2025-08-21T12:34:56Z"`.

### 2.2 Optional fields

* `anchor` (string) — logical section identifier within the resource (e.g., `"tiers"`). When present, the event targets that section.
* `checksum` (string) — fingerprint of the current representation of the event's target (resource or section).
* `note` (string) — short, machine-friendly hint about the change (e.g., `"text-edit"`, `"policy-change"`).

```json
{"id":"01J3Z6A6M7A0CAX","action":"create","url":"https://docs.example/guide/start","time":"2025-08-21T09:00:00Z"}
{"id":"01J3Z6B1KQX6XYT","action":"update","url":"https://docs.example/pricing#tiers","time":"2025-08-21T10:15:00Z","anchor":"tiers","checksum":"sha256:def456","note":"text-edit"}
{"id":"01J3Z6C8S8B1J2W","action":"delete","url":"https://docs.example/old-api","time":"2025-08-21T12:00:00Z"}
```

## 3. Anchors (Simple Granularity)

**Anchor** = stable, logical section label in the resource (e.g., HTML `id`, Markdown heading `{#id}`, or publisher-defined section key). Anchors enable **section-level invalidation** without standardizing chunk sizes.

* If `anchor` is present, the event refers to that section.
* If absent, the event refers to the entire resource (`url`).

**Why anchors?** They're stable for humans and machines, and they keep v0.1 simple:

* Publishers don't need to publish chunk IDs.
* Consumers can re-fetch and re-process only the **section** that changed.


## 4. Client Behavior (Minimal)

A compliant client SHOULD:

1. Fetch discovery; verify compatible `aicf_version`.
2. Read the NDJSON feed incrementally from the **last seen** `id`.
3. Deduplicate on `id`; persist the latest processed `id`.
4. Use HTTP caching (ETag / Last-Modified) for efficient polling.
5. Apply events:
   * **create / update**
      * Determine the **invalidation boundary**:
         * `anchor` present ⇒ boundary is the section (`url#anchor`).
         * No `anchor` ⇒ boundary is the whole resource (`url`).
      * Optionally compare `checksum` to skip no-op work.
      * Re-fetch only the boundary and **re-process locally** (e.g., re-chunk, re-embed, re-index).
   * **delete**
      * Remove all locally stored data derived from `url`.
      * If a future version uses `delete` with `anchor`, remove data derived from that section only (v0.1 clients MAY treat such events as full-resource delete if unimplemented).

**Note:** This spec does **not** define chunking. Clients that maintain chunked indexes **rebuild the boundary** (section or whole resource) using their own chunking scheme.

## 5. Publisher Behavior (Minimal)

* **Append-only:** past lines in the feed MUST NOT change.
* **Sortable IDs:** ensure `id` ordering is stable and monotonic within a feed.
* **Anchors:** keep anchor labels stable across edits when possible.
* **Checksums:** when feasible, include a checksum of the boundary's current representation to allow clients to skip no-op work.
* **Rotation (optional):** if rotating feeds, keep `self` pointing at the current file and retain enough history for clients to catch up.

## 6. HTTP & Caching

* Publishers SHOULD expose `ETag` and/or `Last-Modified` headers for the feed and for target resources.
* Clients SHOULD use conditional requests and retry safely (reading the same range of events must be idempotent).

## 7. Interoperability

* **Sitemaps** → "what exists"; **AICF** → "what changed." Use both.
* **RSS/Atom** → human news; **AICF** → machine-granular updates (docs, pricing, policies).
* **robots.txt / llms.txt** → access/usage policy; **AICF** → updates channel.
* **MCP (Model Context Protocol)** → AICF feeds can serve as efficient data sources for MCP servers, enabling real-time context updates for AI assistants. 

## 8. JSON Schemas (Minimal)

### 8.1 Discovery

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AICF Discovery v0.1 (Minimal)",
  "type": "object",
  "required": ["aicf_version", "self"],
  "properties": {
    "aicf_version": { "type": "string" },
    "self": { "type": "string", "format": "uri" },
    "license": { "type": "string" },
    "about": { "type": "string", "format": "uri" },
    "ttl_seconds": { "type": "integer", "minimum": 0 }
  },
  "additionalProperties": true
}
```

### 8.2 Event (NDJSON line)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "AICF Event v0.1 (Minimal)",
  "type": "object",
  "required": ["id", "action", "url", "time"],
  "properties": {
    "id": { "type": "string", "minLength": 1 },
    "action": { "type": "string", "enum": ["create", "update", "delete"] },
    "url": { "type": "string", "format": "uri" },
    "time": { "type": "string", "format": "date-time" },
    "anchor": { "type": "string" },
    "checksum": { "type": "string" },
    "note": { "type": "string" }
  },
  "additionalProperties": true
}

```

# 9. Examples (End-to-end)

### Discovery

```json
{
  "aicf_version": "0.1",
  "self": "https://docs.acme.dev/ai-changes.ndjson",
  "license": "CC BY 4.0",
  "ttl_seconds": 120
}

```

### Feed (NDJSON)

```json
{"id":"01J3Z6A6...","action":"create","url":"https://docs.acme.dev/guide#intro","time":"2025-08-21T09:00:00Z"}
{"id":"01J3Z6B1...","action":"update","url":"https://docs.acme.dev/pricing#tiers","time":"2025-08-21T10:15:00Z","anchor":"tiers","checksum":"sha256:8c1...","note":"text-edit"}
{"id":"01J3Z6C8...","action":"delete","url":"https://docs.acme.dev/old-api","time":"2025-08-21T12:00:00Z"}

```

### Client (conceptual flow)

```

onEvent(e):
  if e.action == "delete":
    delete_all_data_for(e.url)
  else:
    boundary = e.anchor ? (e.url + "#" + e.anchor) : e.url
    if e.checksum and checksum_unchanged(boundary, e.checksum):
      return
    content = fetch(boundary)
    reprocess_locally(content)  // re-chunk/re-embed/re-index as you like

```

## 10. Change Log

* **0.1 (Minimal Core):** Discovery + append-only NDJSON feed with `id|action|url|time`; optional `anchor|checksum|note`; client boundary invalidation; no chunk IDs, no vectors, no push, no signatures, no billing.

## 11. What to Add Next (Optional Extensions)

1. **Chunk Hints & Precomputed Vectors**
  * Fields like `chunk_id`, `embed_ref`, `section_checksum` for surgical updates and RAG efficiency.

2. **Push Delivery**
  * Webhook/SSE profiles for near-real-time updates with retry semantics.

3. **Billing Hints**
  * Neutral metering metadata (`plan|unit|size|nonce`) for usage reconciliation.

Each extension should be **opt-in** and discoverable (e.g., by a future `capabilities` array), keeping the core spec small and easy to adopt.

## 12. Contributors & Acknowledgments

* **Lead Author:** Manas Wadhwa 
* **Feedback & Ideas:** [acknowledgments as they come]
* **Contact:** manaswadhwa41@gmail.com



