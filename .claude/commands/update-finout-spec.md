# Update Finout OpenAPI Spec

Fetch the latest Finout API V2 documentation and update `openapi/finout-v2.yaml`, then regenerate the Go client.

## Step 1 — Discover all V2 API pages

Fetch https://docs.finout.io/sitemap.md and extract every URL that belongs to the Finout API V2 section. The known pages are:

- https://docs.finout.io/api/finout-api/finout-api-v2/virtual-tags-api-v2-beta.md
- https://docs.finout.io/api/finout-api/finout-api-v2/virtual-tag-versions-api-v2-beta.md
- https://docs.finout.io/api/finout-api/finout-api-v2/virtual-tag-metadata-api-v2-beta.md
- https://docs.finout.io/api/finout-api/finout-api-v2/query-language-api-v2.md
- https://docs.finout.io/api/finout-api/finout-api-v2/cost-and-usage-api-v2.md
- https://docs.finout.io/api/finout-api/finout-api-v2/filter-object-definition-v2.md

Fetch the sitemap to check for any additions or removals, then fetch each page in parallel. For each page, extract:

- All endpoints: HTTP method, full path, operationId (derive from summary if not explicit)
- All path and query parameters: name, location, type, required, description
- Request body schema: all fields, types, required/optional, constraints
- Response schemas for every status code: all fields, types
- Rate limits (add to endpoint description)
- Any constraints or business rules worth preserving in descriptions

## Step 2 — Diff against the current spec

Read `openapi/finout-v2.yaml`. For each API page fetched:

- **New endpoint**: add it under the correct tag in `paths:` and add any new schemas to `components/schemas:`
- **Changed endpoint**: update the affected path, parameters, request body, or response schemas
- **Removed endpoint**: remove the path and any schemas that are now exclusively referenced by that path
- **New API section** (not yet in the spec): add a new tag, all its paths, and all its schemas

Preserve all existing descriptions, constraints, and comments that are not contradicted by the new documentation. Do not widen existing enum values unless the documentation explicitly lists new values.

## Step 3 — Validate the updated spec

After editing `openapi/finout-v2.yaml`, run:

```bash
python3 -c "
import yaml, re, sys
doc = yaml.safe_load(open('openapi/finout-v2.yaml'))
content = open('openapi/finout-v2.yaml').read()
schemas = set('#/components/schemas/' + k for k in doc['components']['schemas'])
params  = set('#/components/parameters/' + k for k in doc['components']['parameters'])
resps   = set('#/components/responses/' + k for k in doc['components']['responses'])
defined = schemas | params | resps
used = set(r.strip('\"') for r in re.findall(r'\"#/components/[^\"]+\"', content))
missing = used - defined
if missing:
    print('MISSING refs:', missing); sys.exit(1)
print(f'OK — {len(used)} refs, {len(doc[\"paths\"])} path items, {len(doc[\"components\"][\"schemas\"])} schemas')
"
```

Fix any errors before proceeding.

## Step 4 — Regenerate the Go client

```bash
go generate ./client/
go build ./client/
```

Fix any build errors. Common causes:
- A new `oneOf` schema needs `FilterCondition`-style union helper methods — oapi-codegen generates these automatically if the spec uses `oneOf` correctly.
- A new runtime import may be needed: `go get github.com/oapi-codegen/runtime@latest`.

## Step 5 — Report

List every change made:
- Endpoints added, modified, or removed
- Schemas added, modified, or removed
- Whether the Go client regenerated and compiled cleanly
