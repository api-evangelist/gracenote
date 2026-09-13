---
name: gracenote-tv-listings-with-onconnect
description: >-
  Build a TV guide or listings experience on the Gracenote OnConnect Lookup APIs — find the viewer's
  lineup, get the channel list, pull the grid, and drill into a program, series, movie or celebrity.
generated: '2026-09-12'
method: generated
source: openapi/gracenote-onconnect-lookup-apis-openapi.json
api: gracenote:onconnect-lookup-apis
base_url: https://data.tmsapi.com/v1.1
auth: api_key query parameter
operations_note: >-
  The OnConnect specification declares no operationIds for any of its 40 operations, so this skill
  cites each operation by METHOD and path exactly as published. Do not invent operationIds for them.
---

# TV listings with the OnConnect Lookup APIs

40 read operations over schedules, programs, series, movies, sports, stations, lineups, theatres and
celebrities. Authenticate by appending `api_key` as a query parameter. The gateway returns
`403 ERR_403_DEVELOPER_INACTIVE` if the key is missing or the account is not active.

## The grid, end to end

1. **Find the viewer's lineup.** `GET /v1.1/lineups` — the lineup is the channel package for a
   provider in a place. Everything schedule-shaped hangs off it.
2. **Confirm it.** `GET /v1.1/lineups/{lineupId}`.
3. **Get the channels.** `GET /v1.1/lineups/{lineupId}/channels`.
4. **Get the grid.** `GET /v1.1/lineups/{lineupId}/grid` — the airings matrix a TV guide renders.
5. **Single-channel view.** `GET /v1.1/stations/{stationId}/airings`, with
   `GET /v1.1/stations/search` and `GET /v1.1/stations/{stationId}` for lookup and detail.

## Drilling into a title

- `GET /v1.1/programs/search` → `GET /v1.1/programs/{tmsId}` → `GET /v1.1/programs/{tmsId}/airings`.
- Artwork: `GET /v1.1/programs/{resourceId}/images`. Genres: `GET /v1.1/programs/genres`.
- Series: `GET /v1.1/series/{seriesId}`, `/airings`, `/episodes`.
- Movies on TV: `GET /v1.1/movies/airings`, `GET /v1.1/movies/{rootId}/airings`, and
  `GET /v1.1/movies/{rootId}/versions` for alternate cuts and languages.
- Celebrities: `GET /v1.1/celebs/{personId}`, `/airings`, `/images`, plus
  `GET /v1.1/celebs/talkShowAirings` for who is on which talk show on a given day.

## Editorial and discovery surfaces

These are the operations that make a guide feel curated rather than mechanical:

- `GET /v1.1/programs/newShowAirings` — new shows airing.
- `GET /v1.1/programs/newShowsLastWeek` — what premiered in the past week.
- `GET /v1.1/programs/advancePlanner` — forward-looking programming.
- `GET /v1.1/movies/futureReleases` — upcoming theatrical releases.

## Theatrical

`GET /v1.1/movies/showings` (what is playing near a location),
`GET /v1.1/theatres` / `GET /v1.1/theatres/{theatreId}` / `GET /v1.1/theatres/{theatreId}/showings`,
and `GET /v1.1/movies/{movieId}/showings` for one film.

## Sports

`GET /v1.1/sports/{sportsId}` and `/events/airings`, `/non-events/airings`;
organizations, universities and teams via `GET /v1.1/sports/organizations/{organizationId}`,
`GET /v1.1/sports/universities`, `GET /v1.1/sports/universities/{universityId}`,
`GET /v1.1/sports/teams/{teamBrandId}` and the matching `/airings` variants.

> For deep sports data — standings, stats, play-by-play — use the Global Sports Data Lookup API
> instead. OnConnect answers "is it on TV"; Global Sports Data answers "what happened in it".

## Conventions

- Pagination is `offset`/`limit`; keep `limit` at or below 1000.
- The spec declares `200` responses only. Handle `403` (inactive key or unentitled feature) and `429`
  defensively; there are no rate-limit response headers to read.
- Image sizing and aspect are request parameters, not separate operations.
- This API is the legacy-gateway product. Gracenote moved its documentation to
  https://devportal.gracenote.com/catalog/onconnect on 2026-09-08; the `data.tmsapi.com/v1.1` host is
  still the documented base.
