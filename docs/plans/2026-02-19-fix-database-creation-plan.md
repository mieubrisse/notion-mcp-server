# Fix Database Creation Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Restore working database creation in the Notion MCP server by adding the correct `POST /v1/databases` endpoint to the OpenAPI spec, fixing the misleading `create-a-data-source` description, updating the README to document fork divergences, and identifying upstream GitHub issues affected by this bug.

**Architecture:** All MCP tools are auto-generated from `scripts/notion-openapi.json` at server startup by `OpenAPIToMCPConverter` in `src/openapi-mcp-server/openapi/parser.ts`. No TypeScript changes are needed — editing the JSON spec is sufficient to add/fix tools. The proxy in `src/openapi-mcp-server/mcp/proxy.ts` picks up all changes automatically.

**Tech Stack:** TypeScript, Node.js, MCP SDK, OpenAPI 3.0, `gh` CLI for GitHub issue search.

**Design doc:** `docs/plans/2026-02-19-fix-database-creation-design.md`

---

### Task 1: Add `POST /v1/databases` to the OpenAPI spec

**Context:** The spec at `scripts/notion-openapi.json` is missing the `POST /v1/databases` endpoint. In the Notion API 2025-09-03, this is the correct endpoint for creating a new database. Properties for the initial data source must be nested under `initial_data_source.properties` — not at the top level (which was the old pre-2025-09-03 format).

**Files:**
- Modify: `scripts/notion-openapi.json`

**Step 1: Locate the insertion point**

Open `scripts/notion-openapi.json`. Find the `"paths"` object. Add a new key `"/v1/databases"` alongside the existing path entries (e.g., after `"/v1/pages"`).

**Step 2: Add the new path entry**

In the `"paths"` object, add:

```json
"/v1/databases": {
  "post": {
    "summary": "Create a database",
    "description": "Creates a new database as a subpage in the specified parent page, with an initial data source whose property schema is defined by initial_data_source.properties. To add a data source to an existing database instead, use create-a-data-source.",
    "operationId": "create-a-database",
    "tags": ["Databases"],
    "parameters": [
      {
        "$ref": "#/components/parameters/notionVersion"
      }
    ],
    "requestBody": {
      "content": {
        "application/json": {
          "schema": {
            "type": "object",
            "required": ["parent"],
            "properties": {
              "parent": {
                "$ref": "#/components/schemas/pageIdParentRequest"
              },
              "title": {
                "type": "array",
                "description": "Name of the database.",
                "items": {
                  "$ref": "#/components/schemas/richTextRequest"
                },
                "maxItems": 100
              },
              "description": {
                "type": "array",
                "description": "Description of the database.",
                "items": {
                  "$ref": "#/components/schemas/richTextRequest"
                },
                "maxItems": 100
              },
              "is_inline": {
                "type": "boolean",
                "description": "Whether the database is displayed inline in its parent page. Defaults to false."
              },
              "icon": {
                "type": "object",
                "description": "Icon for the database. Emoji example: {\"type\": \"emoji\", \"emoji\": \"📋\"}. External image example: {\"type\": \"external\", \"external\": {\"url\": \"https://example.com/icon.png\"}}."
              },
              "cover": {
                "type": "object",
                "description": "Cover image for the database. External image example: {\"type\": \"external\", \"external\": {\"url\": \"https://example.com/cover.png\"}}."
              },
              "initial_data_source": {
                "type": "object",
                "description": "Configuration for the database's initial data source.",
                "properties": {
                  "properties": {
                    "type": "object",
                    "description": "Property schema for the initial data source. Keys are property names. Each database must have exactly one 'title' property. Supported types include: title, rich_text, number, select, multi_select, status, date, checkbox, url, email, phone_number, people, files, relation, rollup, formula, created_time, created_by, last_edited_time, last_edited_by. Examples: {\"Name\": {\"title\": {}}, \"Status\": {\"select\": {\"options\": [{\"name\": \"Active\", \"color\": \"green\"}, {\"name\": \"Done\", \"color\": \"blue\"}]}}, \"Due Date\": {\"date\": {}}, \"Count\": {\"number\": {\"format\": \"number\"}}, \"Notes\": {\"rich_text\": {}}, \"Done\": {\"checkbox\": {}}}."
                  }
                }
              }
            }
          }
        }
      }
    },
    "responses": {
      "200": {
        "description": "Successful response",
        "content": {
          "application/json": {
            "schema": {
              "type": "object"
            }
          }
        }
      },
      "400": {
        "description": "Bad request",
        "content": {
          "application/json": {
            "schema": {
              "type": "object",
              "properties": {
                "object": {"type": "string", "example": "error"},
                "status": {"type": "integer", "example": 400},
                "code": {"type": "string"},
                "message": {"type": "string"}
              }
            }
          }
        }
      }
    },
    "deprecated": false,
    "security": []
  }
}
```

**Step 3: Verify the JSON is valid**

Run:
```
node -e "JSON.parse(require('fs').readFileSync('scripts/notion-openapi.json', 'utf8')); console.log('Valid JSON')"
```

Expected output: `Valid JSON`

If you get a parse error, fix it before continuing.

**Step 4: Commit**

```
git add scripts/notion-openapi.json
git commit -m "fix: add create-a-database endpoint with correct 2025-09-03 schema"
git push
```

---

### Task 2: Fix the `create-a-data-source` description

**Context:** The current description says "Create a new data source (database)" which is wrong — this endpoint adds a data source to an *existing* database. AI agents read these descriptions to decide which tool to call, so the wrong description causes them to call the wrong endpoint.

**Files:**
- Modify: `scripts/notion-openapi.json`

**Step 1: Find the description field**

In `scripts/notion-openapi.json`, find the `POST /v1/data_sources` entry. It currently reads:

```json
"description": "Create a new data source (database)",
```

**Step 2: Update the description**

Change it to:

```json
"description": "Adds an additional data source (with its own property schema and rows) to an existing database. To create a brand-new database, use create-a-database instead.",
```

**Step 3: Verify the JSON is valid**

Run:
```
node -e "JSON.parse(require('fs').readFileSync('scripts/notion-openapi.json', 'utf8')); console.log('Valid JSON')"
```

Expected output: `Valid JSON`

**Step 4: Commit**

```
git add scripts/notion-openapi.json
git commit -m "fix: correct create-a-data-source description to clarify it requires an existing database"
git push
```

---

### Task 3: Build and verify

**Context:** After modifying the spec, make sure the TypeScript project still builds cleanly. The OpenAPI→MCP converter will parse the updated spec at startup, so a clean build is a good smoke test.

**Files:** None modified.

**Step 1: Install dependencies (if needed)**

```
npm install
```

**Step 2: Run the build**

```
npm run build
```

Expected: Build completes with no errors. If there are errors, investigate — they likely indicate a schema issue in the spec you added.

**Step 3: Verify tool names in output**

After the build, run the tests to confirm no regressions:

```
npm test
```

Expected: All tests pass.

**Step 4: Confirm `create-a-database` tool appears**

Start the server in a test mode and list available tools to confirm `create-a-database` now appears. The quickest way is to inspect the spec parse output directly:

```
node -e "
const spec = JSON.parse(require('fs').readFileSync('scripts/notion-openapi.json', 'utf8'));
const paths = Object.keys(spec.paths);
const hasDb = paths.includes('/v1/databases');
console.log('Has /v1/databases path:', hasDb);
const op = spec.paths['/v1/databases']?.post;
console.log('operationId:', op?.operationId);
console.log('has initial_data_source:', !!op?.requestBody?.content?.['application/json']?.schema?.properties?.initial_data_source);
"
```

Expected output:
```
Has /v1/databases path: true
operationId: create-a-database
has initial_data_source: true
```

---

### Task 4: Integration test

**Context:** Verify the end-to-end flow works with a real Notion API call. This requires a live Notion integration token and a parent page ID that the integration has access to.

**This task is manual — performed by the user, not automated.**

**Step 1: Configure the server**

Ensure `NOTION_TOKEN` is set to a valid Notion integration token.

**Step 2: Run the server**

```
npx notion-mcp-server
```

Or if running from source:
```
npm run build && node dist/index.js
```

**Step 3: Call create-a-database**

Using your MCP client (Claude Desktop, Cursor, etc.), invoke the `create-a-database` tool with:

```json
{
  "parent": { "page_id": "<a-valid-page-id>" },
  "title": [{ "text": { "content": "Test Database" } }],
  "initial_data_source": {
    "properties": {
      "Name": { "title": {} },
      "Status": {
        "select": {
          "options": [
            { "name": "Active", "color": "green" },
            { "name": "Done", "color": "blue" }
          ]
        }
      },
      "Due Date": { "date": {} }
    }
  }
}
```

**Expected:** A new Notion database named "Test Database" with Name (title), Status (select), and Due Date (date) columns appears as a subpage of the specified parent page.

**If it fails:** Check the error message carefully. A `validation_error` from Notion usually means a malformed property schema. A `401` means the token is wrong. A `404` means the parent page ID doesn't exist or the integration doesn't have access.

**Step 5: Clean up**

Delete the test database from Notion after confirming it works.

---

### Task 5: Update README fork notice

**Context:** The blockquote notice at the top of `README.md` explains why this fork exists. It should also document the specific divergence from upstream — particularly the broken `create-a-database` behavior and the fix applied here.

**Files:**
- Modify: `README.md` (lines 1–12, the blockquote at the top)

**Step 1: Find the existing notice**

The current notice reads (approximately):

```markdown
> # 🚨 READ THIS 🚨
>
> This is a fork of [the once-official Notion MCP server](https://github.com/makenotion/notion-mcp-server). The Notion team is no longer maintaining theirs because they're trying to push folks to the remote MCP server.
>
> Unfortunately, the remote MCP Notion server [disallows databases unless you have the Enterprise plan](...). Therefore, I, [mieubrisse](https://github.com/mieubrisse), am maintaining this fork.
>
> I'm publishing this as a public good since I'm going to fix bugs in my own workflows anyways, but **I don't have time for anything except bugfixes.**
>
> Concretely, this means:
> - I'll fix my own bugs and publish the here
> - I'll occasionally review bugfix PRs from other folks
> - I won't consider features; please don't file them
```

**Step 2: Add a "How this fork differs from upstream" section**

Append the following inside the blockquote, after the existing content:

```markdown
>
> **How this fork differs from upstream:**
>
> - **`create-a-database` is restored and fixed.** The upstream v2.0.0 migration removed `POST /v1/databases` and replaced it with `POST /v1/data_sources`, but these endpoints do different things in the Notion API 2025-09-03. `POST /v1/data_sources` adds a data source to an _existing_ database — it cannot create new databases. The Notion API returns a hard error if you try. This fork adds `POST /v1/databases` back with the correct request schema (properties belong under `initial_data_source.properties`, not at the top level).
```

**Step 3: Verify formatting renders correctly**

Preview the Markdown to make sure the blockquote renders without broken formatting.

**Step 4: Commit**

```
git add README.md
git commit -m "docs: update fork notice to document database creation fix vs upstream"
git push
```

---

### Task 6: Identify affected upstream GitHub issues

**Context:** Search [github.com/makenotion/notion-mcp-server](https://github.com/makenotion/notion-mcp-server) for open issues where users are hitting the broken database creation behavior. Produce a list of issue URLs only — no commenting, no action.

**Files:** None modified.

**Step 1: Search for database creation issues**

Run each of the following `gh` search commands and collect matching issue URLs:

```
gh issue list --repo makenotion/notion-mcp-server --state open --search "create database" --json number,title,url
```

```
gh issue list --repo makenotion/notion-mcp-server --state open --search "create-a-database" --json number,title,url
```

```
gh issue list --repo makenotion/notion-mcp-server --state open --search "create-a-data-source" --json number,title,url
```

```
gh issue list --repo makenotion/notion-mcp-server --state open --search "Creating new databases with data sources is not supported" --json number,title,url
```

```
gh issue list --repo makenotion/notion-mcp-server --state open --search "initial_data_source" --json number,title,url
```

**Step 2: Filter to relevant issues**

Review the results and keep only issues that are clearly about **database creation failing** (not general database query/update issues). Discard anything unrelated to the `create-a-database` / `create-a-data-source` confusion.

**Step 3: Output the list**

Print the final list of relevant issue URLs. No further action — commenting is deferred until the fix is confirmed working and published.
