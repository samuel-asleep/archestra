# Implementation Plan: Fix `archestra__get_mcp_gateway` (and related get tools)

## Problem Location

**File:** `platform/backend/src/archestra-mcp-server.ts`, lines ~1714–1735

The `archestra__get_mcp_gateway`, `archestra__get_agent`, and `archestra__get_llm_proxy` tools all share the same handler. When the `id` parameter is provided, it is passed directly to `AgentModel.findById(id)`:

```typescript
if (id) {
  record = await AgentModel.findById(id);  // ❌ No UUID validation
}
```

`AgentModel.findById` runs the following SQL:
```sql
SELECT ... FROM "agents" ... WHERE "agents"."id" = $1
-- params: "n8n workflow: Grafana exporter"  ← NOT a UUID!
```

Because `agents.id` is a `uuid` column in PostgreSQL, passing a human-readable name string causes the query to fail with an error. The user sees:

> `"An unexpected error occurred. Please try again."`

This happens when an LLM (or user) passes a **name** as the `id` field instead of a valid UUID — e.g., `{ "id": "n8n workflow: Grafana exporter" }`.

---

## Root Cause

No validation is performed on the `id` argument before it is used in a UUID-keyed database lookup. The tool's description already documents support for both `id` and `name` parameters, and a name-search fallback via `AgentModel.findAllPaginated` is already implemented — but it is only reached when `id` is absent entirely.

---

## Proposed Fix

### 1. Add a UUID validation utility

**File:** `platform/backend/src/utils/db.ts`

```typescript
const UUID_REGEX =
  /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

export function isValidUUID(value: string): boolean {
  return UUID_REGEX.test(value);
}
```

### 2. Use UUID validation in the get-entity handler

**File:** `platform/backend/src/archestra-mcp-server.ts`

Import `isValidUUID` and update the handler logic so that a non-UUID `id` falls back to name search:

```typescript
// Before (broken):
if (id) {
  record = await AgentModel.findById(id);
} else if (name) {
  // name search ...
}

// After (fixed):
if (id && isValidUUID(id)) {
  record = await AgentModel.findById(id);
} else if (id || name) {
  // If id is not a valid UUID, treat it as a name search.
  const searchName = (id && !isValidUUID(id) ? id : name) as string;
  const results = await AgentModel.findAllPaginated(
    { limit: 1, offset: 0 },
    undefined,
    { name: searchName, agentType: expectedType, scope: "personal",
      authorIds: context.userId ? [context.userId] : [] },
    context.userId,
    true,
  );
  if (results.data.length > 0) {
    record = results.data[0];
  }
}
```

---

## Affected Tools

All three tools share the same handler block and are fixed by this single change:

| Tool | Type filter |
|------|-------------|
| `archestra__get_mcp_gateway` | `mcp_gateway` |
| `archestra__get_agent` | `agent` |
| `archestra__get_llm_proxy` | `llm_proxy` |

---

## Tests Added

New test cases in `platform/backend/src/archestra-mcp-server.test.ts`:

- `get_mcp_gateway`: error on missing id and name
- `get_mcp_gateway`: find by valid UUID id
- `get_mcp_gateway`: find by name when non-UUID string is passed as `id` *(reproduces the reported bug)*
- `get_mcp_gateway`: find by `name` parameter
- `get_mcp_gateway`: error when gateway not found
- `get_agent`: error on missing id and name
- `get_agent`: find by valid UUID id
- `get_agent`: find by name when non-UUID string is passed as `id`
- `get_llm_proxy`: error on missing id and name
- `get_llm_proxy`: find by name when non-UUID string is passed as `id`
