# Koii Server: Promo Sends Automation + Config Overhaul

## Context

The server was originally built for the Trass Notion workspace. That workspace is no longer accessible, so all hardcoded DB IDs are stale. We need to:
1. Make the server workspace-agnostic by moving all DB IDs to env vars
2. Add a new `/webhook/promo-sends` endpoint for the Promo Sends automation
3. Clean up references to the old workspace

## Phase 1: Config Overhaul

**File: `server.js`**
- Move hardcoded `PRODUCT_WORKFLOWS_DB_ID` and `STORIES_DB_ID` to `process.env` reads
- Add a comment block above the workflow-copy endpoint noting it was built for the Trass Notion workspace and needs new DB IDs + property name review before reuse
- Keep all workflow-copy code intact and functional — just env-driven now

**File: `env.example`** — update to include all DB IDs:
```
# Notion API
NOTION_API_TOKEN=your_notion_integration_token_here

# Server
PORT=3000

# DB IDs — Workflow Copy (built for Trass workspace, needs reconfiguration)
PRODUCT_WORKFLOWS_DB_ID=
STORIES_DB_ID=

# DB IDs — Promo Sends
PROMO_STORIES_DB_ID=
PROMO_CHANNELS_DB_ID=
PROMO_SENDS_DB_ID=
```

**File: `package.json`** — rename from `product-tools-render-server` to `koii-server`

**File: `render.yaml`** — update service name from `product-tools-render-server` to `koii-server`

## Phase 2: New Endpoint — `POST /webhook/promo-sends`

### Flow

1. **Receive webhook** — extract `storyId` from the payload (same flexible extraction pattern as existing endpoint: `req.body.storyId`, `req.body.data.id`, etc.)
2. **Fetch story page** — call `notion.pages.retrieve({ page_id: storyId })`, extract project relation(s) from a "Projects" property (use multi-name candidate pattern: `Projects`, `Project`, `📁 Projects`, etc.)
3. **Query Channels DB** — `notion.databases.query()` on `PROMO_CHANNELS_DB_ID`, filter for channels whose Projects relation overlaps with the story's project IDs. Since Notion API doesn't support "relation contains any of [list]", we'll query for each project ID separately and deduplicate.
4. **Check for existing Promo Sends** — query `PROMO_SENDS_DB_ID` filtering by the story's Event relation to find already-created sends. Collect existing channel IDs to skip.
5. **Bulk-create Promo Send pages** — for each new channel, create a page in the Promo Sends DB with:
   - `Event` relation → story page ID
   - `Channel` relation → channel page ID
   - `Sent` checkbox → `false`
6. **Embed linked view in story page** — append a linked database view block of the Promo Sends DB to the story page, filtered by `Event` = this story. This gives the story its own "promo home base" checklist while the master Promo Sends DB serves as cross-event tracking. Use `notion.blocks.children.append()` with a `child_database` or `link_to_database` block type. Before appending, check if a linked view already exists on the page (to avoid duplicates on re-trigger).
7. **Return summary** — respond with count of created vs skipped sends, plus whether the linked view was added

### Helper Functions (new)

- `getStoryProjects(storyId)` — fetches story, returns array of project page IDs
- `getChannelsForProjects(projectIds)` — queries Channels DB, returns deduplicated list of matching channels
- `getExistingPromoSends(storyId)` — queries Promo Sends DB for existing rows linked to this story, returns set of channel IDs already present
- `createPromoSends(storyId, channels)` — bulk creates Promo Send pages, returns results
- `embedPromoSendsView(storyId)` — appends a linked database view of Promo Sends (filtered to this story) into the story page; skips if one already exists

### Error Handling

Follow existing patterns:
- Validate payload (return 400 if no story ID)
- Handle Notion API errors with specific messages (unauthorized, not_found, validation_error)
- Continue creating remaining sends if one fails
- Return partial success results

## Phase 3: Cleanup

- Update `README.md` to document both endpoints (noting the workflow-copy one is legacy/Trass and needs reconfiguration) and the new env vars
- Update test expectations if needed

## Files to Modify

| File | Changes |
|------|---------|
| `server.js` | Move DB IDs to env vars, add `/webhook/promo-sends` endpoint + helpers |
| `env.example` | Add all new env vars |
| `package.json` | Rename to `koii-server` |
| `render.yaml` | Rename service |
| `README.md` | Document new endpoint and config |

## Important Context

The original project lives at `/Users/kb20250422/Documents/dev/trass/` but the actual server files (server.js, package.json, render.yaml, env.example, README.md) were not found in that directory during planning. The project may need to be bootstrapped fresh or the existing files may be in a different location/branch. Check git history or ask the user for the original source before starting implementation.

## Verification

1. `npm start` — server boots without errors (with placeholder env vars it should still start, just fail on API calls)
2. `curl http://localhost:3000/health` — returns OK
3. `curl -X POST http://localhost:3000/webhook/promo-sends -H "Content-Type: application/json" -d '{"storyId":"test-id"}'` — returns a meaningful error (not a crash) since the ID is fake
4. Once real DB IDs and Notion token are configured: trigger from a Notion button and verify:
   - Promo Send rows are created with correct Event/Channel relations and Sent=unchecked
   - No duplicates on re-trigger (same story ID should skip already-created channels)
   - A linked database view of Promo Sends appears inside the story page, filtered to that story
   - Re-triggering does not add a second linked view
