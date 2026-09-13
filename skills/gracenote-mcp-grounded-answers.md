---
name: gracenote-mcp-grounded-answers
description: >-
  Ground an LLM's answers about movies, TV and sports in Gracenote's verified metadata using the
  Video and Sports MCP servers — connecting, the identity-first call order, and the per-tool input
  ceilings that will otherwise silently truncate your request.
generated: '2026-09-12'
method: generated
source: https://devportal.gracenote.com/video/video-mcp-server/tool-reference + https://devportal.gracenote.com/sports/sports-mcp-server/tool-reference
api: gracenote:video-mcp-server, gracenote:sports-mcp-server
endpoints:
  - https://video.mcp.gracenote.com/mcp
  - https://sports.mcp.gracenote.com/mcp
auth: OAuth 2.0
operations:
  - gnv_resolve_entities
  - gnv_get_availability
  - gnv_get_root
  - gnv_get_tmsid
  - gnv_get_images
  - gnv_get_cast
  - gnv_get_awards
  - gnv_get_ratings
  - gnv_web_search
  - gns_resolve_entities
  - gns_discover_events
  - gns_get_match_info
  - gns_get_match_summary
  - gns_get_match_lineups
  - gns_where_to_watch
  - gns_get_team
  - gns_get_athlete
  - gns_get_league
  - gns_get_standings
  - gns_get_images
---

# Grounded answers from the Gracenote MCP servers

Two remote MCP servers, one for video and one for sports. Both speak JSON-RPC over streamable HTTP and
both are OAuth-protected.

## Connecting

An anonymous `tools/list` returns `401` with a `WWW-Authenticate: Bearer` challenge carrying
`resource_metadata`. A compliant MCP client follows that to
`/.well-known/oauth-protected-resource/mcp`, then to `/.well-known/oauth-authorization-server`, and
registers itself dynamically at `/register`. PKCE `S256` is supported.

For server-to-server use, Gracenote documents a client-credentials grant against
`https://auth.mcp.gracenote.com/oauth2/token`. The access token is valid for **one hour**, so a
long-running process must refresh. Gracenote states the server does not enforce scopes on M2M tokens.

Interactive testing is done with MCP Inspector against the production endpoint — there is no sandbox.

## Video: always resolve first

`gnv_resolve_entities` turns a human title into a Gracenote identifier. Every other video tool takes
that identifier, not a name.

```
gnv_resolve_entities  →  tmsId  →  gnv_get_tmsid / gnv_get_availability / gnv_get_images
                                   gnv_get_cast / gnv_get_awards / gnv_get_ratings
```

Input ceilings that matter, because exceeding them is not an error you will notice:

| Tool | Ceiling |
|---|---|
| `gnv_resolve_entities` | 5 entities per request; 3 actor names per entity |
| `gnv_get_availability` | **1 TMSID** per request; `countryCode` is ISO 3166-1 alpha-3 |
| `gnv_get_tmsid`, `gnv_get_images`, `gnv_get_cast`, `gnv_get_awards`, `gnv_get_ratings` | **1 TMSID** each |
| `gnv_get_root` | 1 RootID |
| `gnv_web_search` | query max 400 characters; 1–5 results |

Batch enrichment therefore means N calls, not one call with N ids. Plan for it.

`gnv_get_root` is the tool people miss: it takes a RootID and returns the alternate-language versions
of the same title. If a user asks about a film in a language other than the one you resolved in, go
through the root rather than re-resolving.

`gnv_web_search` is a deliberate pre-processing step — use it to sharpen a vague query *before*
`gnv_resolve_entities`, not as a substitute for it.

**Deprecation:** the unprefixed names `resolve_entities` and `get_availability` still work but
Gracenote states they will be removed. Use the `gnv_` forms.

## Sports: identity first, then exactly one Step 3 tool

The Sports server enforces a three-step architecture and says so explicitly.

1. **Resolve.** `gns_resolve_entities` converts a team, athlete, league or competition name into a
   Gracenote ID. Always called first. No other tool accepts a raw name.
2. **Discover.** `gns_discover_events` takes the resolved ID plus date and status filters and returns
   `matchId`s.
3. **Answer with one tool.** Gracenote's own guidance: *do not call multiple Step 3 tools for the same
   query.*

   | User intent | Tool |
   |---|---|
   | Score, result, winner, status, venue, timing | `gns_get_match_info` (the default when intent is ambiguous) |
   | Recap, key moments, highlights | `gns_get_match_summary` |
   | Where to watch, channel, streaming | `gns_where_to_watch` |
   | Starters, lineup, roster for a match | `gns_get_match_lineups` |

Profile tools take a Step 1 identifier directly and skip discovery: `gns_get_team`,
`gns_get_athlete`, `gns_get_league`, `gns_get_standings`, `gns_get_images`.

## What these servers will not do

Both are read-only. There is no write tool on either server, and the one Gracenote API that does
write — GN IDS — has no MCP surface at all. An agent cannot publish, update or delete anything
through MCP. If your workflow needs to write, it needs the REST API and a human in the loop; see
`skills/gracenote-publish-a-movie-to-gn-ids.md`.

Gaps worth knowing before you promise a user an answer: there is no awards operation in the REST
specs, no sports imagery operation in the Lookup API, and sports broadcast availability lives only in
`gns_where_to_watch` — so for those four capabilities the MCP server is the *only* way in, and if it
is down there is no REST fallback.
