---
name: gracenote-sports-schedule-and-standings
description: >-
  Answer "when do they play, who won, and where do they stand" against the Gracenote Global Sports
  Data Lookup API — the sport → league → season → match resolution chain, using the API's real
  operationIds.
generated: '2026-09-12'
method: generated
source: openapi/gracenote-global-sports-data-lookup-api-openapi.json
api: gracenote:global-sports-data-lookup-api
base_url: https://api.sports.gracenote.com/gns-api
auth: API key
operations:
  - Sports
  - SportInfo
  - LeaguesbySport
  - Leagues
  - LeagueInfo
  - LeagueSeasonsbyLeague
  - LeagueSeasonInfo
  - StructurebyLeague
  - TeamsbyLeague
  - TeamsbyLeagueSeason
  - TeamInfo
  - TeamRosterbyLeagueandTeam
  - ScheduleandResults
  - ScheduleandResultsbyLeague
  - ScheduleandResultsbyTeam
  - MatchInfo
  - LineupsbyMatch
  - ActionsbyMatch
  - TeamStatsbyMatch
  - PersonStatsbyMatch
  - PreviousResultsbyMatch
  - StandingsbyLeague
  - StandingsbyLeagueSeason
  - LeagueSeasonandOverallClassificationsbyLeague
  - PersonInfo
  - VenueInfo
---

# Sports schedules, results and standings

Gracenote's sports data is identifier-first. Nothing in this API accepts a team name, a league name,
or a date range in isolation — you resolve down the hierarchy to an ID, then ask the question.

## The hierarchy

```
Sport ──► Series ──► SeriesSeason
  └─────► League ──► LeagueSeason ──► Match
                        └─► Team ──► Person
                                      Venue
```

## Steps

1. **Find the sport.** `Sports` (`GET /gsd/lookup/v1/sports`) lists them; `SportInfo`
   (`GET /gsd/lookup/v1/sports/{sportId}`) describes one.

2. **Find the competition.** `LeaguesbySport`
   (`GET /gsd/lookup/v1/sports/{sportId}/leagues`) for league sports, or `SeriesbySport` for
   tour/series sports like tennis and cycling. `Leagues` lists them all if you already know what
   you want. `LeagueInfo` gives you the static profile.

3. **Pick the season.** `LeagueSeasonsbyLeague`
   (`GET /gsd/lookup/v1/leagues/{leagueId}/league-seasons`). Almost every downstream question is
   season-scoped — asking a league-level question when you mean "this season" is the most common way
   to get an answer that is technically correct and useless.

4. **Ask the question.**

   | Question | Operation |
   |---|---|
   | When do they play? | `ScheduleandResultsbyLeague`, `ScheduleandResultsbyTeam`, or `ScheduleandResults` |
   | Who won this match? | `MatchInfo` (`GET /gsd/lookup/v1/matches/{matchId}`) |
   | Who started? | `LineupsbyMatch` |
   | What happened in it? | `ActionsbyMatch`, plus `TeamStatsbyMatch` and `PersonStatsbyMatch` |
   | Where do they stand? | `StandingsbyLeagueSeason` (season-scoped) or `StandingsbyLeague` |
   | Who is on the roster? | `TeamRosterbyLeagueandTeam` or `TeamRosterbyLeagueSeasonandTeam` |
   | Tell me about this player | `PersonInfo` |
   | Where is it played? | `VenueInfo` |

5. **Narrow within a league when the league has structure.** `StructurebyLeague` tells you whether the
   league has tiers, conferences or divisions. If it does, the tier/conference/division variants exist
   for both schedule and standings — `StandingsbyLeagueandConference`,
   `ScheduleandResultsbyLeagueandDivision`, and so on. Use them rather than filtering a whole-league
   response client-side.

6. **Non-league formats.** Tours, stage races and individual events use the classification/overall/
   phase/intermediate family instead of standings: `ResultsbyPhase`, `ClassificationsbyOverall`,
   `IntermediatesbyPhase`. `LeagueOverviewbySeries` is the bridge from a series into that model.

## Conventions

- Pagination is `offset`/`limit`. Gracenote recommends a maximum `limit` of 1000.
- Every operation here is a `GET`. There is no write surface on this API — the Global Sports Data
  Update API is a separate contract.
- This spec declares `200` responses only. Handle `401`, `403` and `429` defensively even though the
  contract does not describe them; see `errors/gracenote-problem-types.yml`.
- Rate limits are contractual and unpublished, and there are no rate-limit response headers. Back off
  on `429` with exponential delay.

## When to use the MCP server instead

If the consumer is an LLM rather than your own code, the Sports MCP Server
(`https://sports.mcp.gracenote.com/mcp`) wraps this same data in eleven tools and enforces the
resolution order for you. See `skills/gracenote-mcp-grounded-answers.md`.
