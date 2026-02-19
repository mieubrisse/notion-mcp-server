Fix: Restore Correct Database Creation in Notion MCP Server
============================================================

Date: 2026-02-19
Status: Approved


Background
----------

This repo is a fork of the once-official [makenotion/notion-mcp-server](https://github.com/makenotion/notion-mcp-server). The upstream server's v2.0.0 migration to Notion API version `2025-09-03` introduced a broken database creation tool.

### What the upstream migration got wrong

The Notion `2025-09-03` API introduced a new data model that separates **databases** (containers) from **data sources** (tables within a database — a single database can now have multiple data sources with independent schemas).

The upstream migration:
- Removed `create-a-database` (`POST /v1/databases`)
- Added `create-a-data-source` (`POST /v1/data_sources`) as its replacement

This was incorrect. These two endpoints are **not equivalent**:

| Endpoint | Purpose |
|---|---|
| `POST /v1/databases` | Creates a new database AND its initial data source |
| `POST /v1/data_sources` | Adds an additional data source to an **existing** database |

When an AI agent calls `create-a-data-source` to create a new database, the Notion API returns:

```
"Creating new databases with data sources is not supported in this endpoint
for API version 2025-09-03 and later. Use the Create Database API instead."
```

Per the [Notion upgrade guide](https://developers.notion.com/docs/upgrade-guide-2025-09-03):
> "Continue to use the Create Database API even after upgrading, when you want
> to create both a database and its initial data source."

### Schema change in 2025-09-03

The `POST /v1/databases` endpoint also changed its schema. Properties for the
initial data source now live under `initial_data_source[properties]` rather
than directly in the request body:

Old format (pre-2025-09-03):
```json
{
  "parent": { "type": "page_id", "page_id": "..." },
  "title": [...],
  "properties": { "Name": { "title": {} } }
}
```

New format (2025-09-03):
```json
{
  "parent": { "type": "page_id", "page_id": "..." },
  "title": [...],
  "initial_data_source": {
    "properties": { "Name": { "title": {} } }
  }
}
```


Scope
-----

This fix covers four areas, in order:

1. OpenAPI spec fix
2. Testing
3. README update
4. Upstream GitHub issue identification


Design
------

### 1. OpenAPI Spec Fix (`scripts/notion-openapi.json`)

**Add `POST /v1/databases`** back to the spec with `operationId: create-a-database`.

Request body schema:

```json
{
  "type": "object",
  "required": ["parent"],
  "properties": {
    "parent": { "$ref": "#/components/schemas/pageIdParentRequest" },
    "title": {
      "type": "array",
      "items": { "$ref": "#/components/schemas/richTextRequest" },
      "maxItems": 100
    },
    "description": {
      "type": "array",
      "items": { "$ref": "#/components/schemas/richTextRequest" },
      "maxItems": 100
    },
    "is_inline": {
      "type": "boolean",
      "description": "Whether the database is displayed inline in its parent page (defaults to false)"
    },
    "icon": {
      "type": "object",
      "description": "Icon for the database. Either an emoji object or an external file object."
    },
    "cover": {
      "type": "object",
      "description": "Cover image for the database. An external file object."
    },
    "initial_data_source": {
      "type": "object",
      "description": "Configuration for the database's initial data source, including its property schema.",
      "properties": {
        "properties": {
          "type": "object",
          "description": "Property schema for the initial data source. Keys are property names, values define the property type (e.g. {\"Name\": {\"title\": {}}, \"Status\": {\"select\": {\"options\": [{\"name\": \"Active\", \"color\": \"green\"}]}}})"
        }
      }
    }
  }
}
```

**Fix `create-a-data-source` description** in `POST /v1/data_sources`:

Change:
> "Create a new data source (database)"

To:
> "Add an additional data source to an existing database. To create a new database, use create-a-database instead."

The OpenAPI-to-MCP proxy auto-discovers both changes at startup — no TypeScript changes required.

### 2. Testing

Before any other steps:

1. **Build check** — `npm run build` must pass with no TypeScript errors
2. **Integration test** — Manually invoke `create-a-database` through the running server with:
   - A valid Notion integration token
   - A valid parent page ID
   - A non-trivial properties schema (title column + at least one other type, e.g. select or date)
   - Confirm a database is created in Notion with the expected columns

The integration test is manual (requires a live Notion token held by the user).

### 3. README Update

Update the fork notice block at the top of `README.md` to:
- Explain that the v2.0.0 upstream migration broke database creation
- Document the root cause (wrong endpoint, wrong schema)
- State that this fork restores correct behavior via `create-a-database`

### 4. Upstream GitHub Issue Identification

Search [github.com/makenotion/notion-mcp-server/issues](https://github.com/makenotion/notion-mcp-server/issues) for open issues where users are hitting the broken database creation behavior. Produce a list of issue URLs only — no commenting, no action taken.

Search terms:
- `create database` / `create-a-database`
- `create-a-data-source`
- `"Creating new databases with data sources is not supported"`
- `validation_error`
- `initial_data_source`

Issue identification happens **after** the fix is confirmed working via testing.


References
----------

- [Notion Create Database API reference](https://developers.notion.com/reference/database-create)
- [Notion Create Data Source API reference](https://developers.notion.com/reference/create-a-data-source)
- [Notion upgrade guide for 2025-09-03](https://developers.notion.com/docs/upgrade-guide-2025-09-03)
