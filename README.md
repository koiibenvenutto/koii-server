# Koii Server

Notion automation server with two webhook endpoints: **Promo Sends** (active) and **Workflow Copy** (legacy).

## Setup

1. Install dependencies:
   ```bash
   npm install
   ```

2. Create `.env` from the template:
   ```bash
   cp env.example .env
   ```

3. Fill in your `.env`:
   ```
   NOTION_API_TOKEN=your_token

   # Promo Sends
   PROMO_STORIES_DB_ID=...
   PROMO_CHANNELS_DB_ID=...
   PROMO_SENDS_DB_ID=...

   # Workflow Copy (legacy — see note below)
   PRODUCT_WORKFLOWS_DB_ID=...
   STORIES_DB_ID=...
   ```

4. Ensure your Notion integration has access to the relevant databases.

## Endpoints

### `POST /webhook/promo-sends`

Creates Promo Send rows for a story based on its project-channel mappings.

**Payload:**
```json
{ "storyId": "<notion-page-id>" }
```

**Flow:**
1. Fetches the story's Projects relation
2. Queries Channels DB for channels matching those projects
3. Skips channels that already have a Promo Send for this story
4. Creates a Promo Send page per new channel (`Event` + `Channel` relations, `Sent = false`)
5. Embeds a linked database view of Promo Sends in the story page (once)

**Response:**
```json
{
  "message": "Promo sends processed",
  "storyId": "...",
  "channelsFound": 5,
  "sendsCreated": 3,
  "sendsSkipped": 2,
  "sendsFailed": 0,
  "linkedViewAdded": true
}
```

### `POST /webhook/notion` (Legacy — Workflow Copy)

> **Note:** This endpoint was built for the Trass Notion workspace. The original DB IDs are stale. Before reuse, set `PRODUCT_WORKFLOWS_DB_ID` and `STORIES_DB_ID` in your `.env` and review property names in the target workspace.

Copies workflow template pages into a Stories database with date translation and dependency resolution.

### `GET /health`

Returns `{ "status": "OK" }`.

### `GET /debug`

Returns recent debug messages and API key status.

## Development

```bash
npm run dev     # nodemon with auto-reload
npm start       # production
```

### Docker

```bash
docker-compose up --build -d
docker-compose logs -f
docker-compose down
```

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `NOTION_API_TOKEN` | Yes | Notion internal integration token |
| `PORT` | No | Server port (default: 3000) |
| `PROMO_CHANNELS_DB_ID` | For promo-sends | Channels database ID |
| `PROMO_SENDS_DB_ID` | For promo-sends | Promo Sends database ID |
| `PROMO_STORIES_DB_ID` | For promo-sends | Stories database ID (promo context) |
| `PRODUCT_WORKFLOWS_DB_ID` | For workflow-copy | Product Workflows database ID |
| `STORIES_DB_ID` | For workflow-copy | Stories database ID (workflow context) |
