# glassdata

glassdata is a small local bridge to the hosted HasData
Glassdoor MCP server, which returns Glassdoor job data
as JSON. It can search job listings for a keyword in a
place, and it can pull one posting in full from its
vacancy URL: the job title, the company and its rating,
where the job sits, the salary estimate, when it was
posted, the complete description, and the stated
qualifications and benefits.

The bridge holds no listings of its own. It answers the
tool list locally, exactly as captured from the live
endpoint, and forwards each tool call upstream,
returning whatever comes back untouched. The captured
surface is two read-only fetches over public postings:
there is no application submission, no employer or
candidate contact data, and no messaging or payment
tool in it.

## Tools

- `hasdata_glassdoor_listing_getJobListings` — search
  Glassdoor listings. Required: `keyword` and
  `location`. Optional: `sort` (`recent` / `relevant`),
  `domain` (which national Glassdoor site to target),
  `datePosted` (only jobs posted within that many
  days), `radius` (search distance in miles),
  `minSalary` and `maxSalary` (yearly pay range),
  `minRating` (lowest company rating), `easyApplyOnly`
  and `remoteOnly` flags, and `nextPageToken` for
  paging — the token carries the original search's
  filters with it. Returns jobs with title, company,
  location, salary estimate, posted date, job URL and
  jobId, plus per-page tokens.
- `hasdata_glassdoor_job_getJobDetails` — fetch one
  posting in full. Required: `url` (the vacancy URL).
  Returns title, company name and rating, location,
  salary estimate, posted date, full description, and
  qualifications and benefits.

## Configuration

- `HASDATA_GLASSDOOR_MCP_URL` (optional) — override
  the upstream endpoint. Defaults to
  `https://mcp.hasdata.com/mcp?apis=glassdoor`. (The
  override hook is this rebuild's own labelled
  reconstruction.)
- `HASDATA_API_KEY` (optional) — when set, every
  upstream request carries an `x-api-key: <key>`
  header, for the caller's own HasData API key. When
  it is not set, no key header is sent. (The header
  name and the env name are pinned by the original's
  registry record; forwarding the key from this env
  var is a labelled reconstruction — the original
  surface was captured keyless.)

### Forwarding mechanics

On the first tool call the client opens a streamable
HTTP session upstream: a JSON-RPC `initialize` request
(protocolVersion `2025-03-26`, clientInfo name
`glassdata`), then a `notifications/initialized`
notification, then the `tools/call` request carrying
the caller's arguments unchanged. When the upstream
answers `initialize` with an `Mcp-Session-Id` header,
that value rides on every later request of the
session, which is reused for the life of the process.
Replies may arrive as plain JSON or as server-sent
events; both are read.

Successful upstream results pass through verbatim,
including any `isError` flag the upstream sets.
Failures — a JSON-RPC error, a non-2xx status, an
unreadable response, or a request that cannot be
completed — come back as an `isError` result whose
text starts with `glassdata error:` (this rendering
is this rebuild's own labelled reconstruction).

## CLI

Each tool is also a subcommand taking one JSON
argument string:

```bash
npx @reneturcios/glassdata hasdata_glassdoor_listing_getJobListings '{"keyword":"software engineer","location":"San Francisco, California"}'
npx @reneturcios/glassdata hasdata_glassdoor_job_getJobDetails '{"url":"https://www.glassdoor.com/job-listing/example"}'
npx @reneturcios/glassdata --version
```

Exit codes: `0` on success, `1` on an `isError`
result, `2` on a usage error.

## MCP server

```bash
npx -y @reneturcios/glassdata mcp
```

MCP client config:

```json
{
  "mcpServers": {
    "glassdata": {
      "command": "npx",
      "args": ["-y", "@reneturcios/glassdata", "mcp"]
    }
  }
}
```

## Docker

```bash
docker run -i ghcr.io/cloudcorprecords/glassdata
docker run --rm ghcr.io/cloudcorprecords/glassdata --version
```

The container's default command is `mcp`, so the first
form is a stdio MCP server; any other arguments reach
the CLI normally.

## Development

```bash
npm install
npm run build
npm test
npm run dev
```

The test suite runs entirely against a stubbed
JSON-RPC upstream — no live network calls.

## License

MIT — see `LICENSE`.
