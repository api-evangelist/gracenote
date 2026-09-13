---
name: gracenote-publish-a-movie-to-gn-ids
description: >-
  Create a movie in the Gracenote GN IDS API and publish it to Gracenote-licensed datasets — the
  full root → version → presentation → publish chain, with the delete paths that reverse each step
  and an explicit warning that the publish step itself has no documented undo.
generated: '2026-09-12'
method: generated
source: openapi/gracenote-gn-ids-api-openapi.json
api: gracenote:gn-ids-api
base_url: https://gnids.gracenote.com/api/v1
auth: GN-APIKEY request header
operations:
  - createMovieRoot
  - createMovieVersion
  - createMoviePresentation
  - createMoviePresentationWithVersion
  - mapMoviePresentationsToVersion
  - createImage
  - mapProgramImages
  - publishPresentation
  - bulkPublishPresentations
  - getPublications
  - getMovieRoot
  - getMoviePresentation
  - deleteMovieRoot
  - deleteMovieVersion
  - deleteMoviePresentation
  - deleteMoviePresentationToVersionMapping
  - unmapProgramImages
  - deleteImage
---

# Publish a movie to Gracenote-licensed datasets

The GN IDS API is the only write surface Gracenote publishes. Everything else in the catalog is
read-only. This skill walks the create-and-publish chain for a movie.

## Before you start

- Authenticate with the `GN-APIKEY` request header. Keys are issued by Gracenote sales, not self-serve.
- **There is no idempotency key on any GN IDS operation.** If a `POST` times out, do not blindly retry
  — call the matching `GET` first (`getMovieRoot`, `getMoviePresentation`) and check whether the object
  already exists. A retry can create a duplicate record.
- **There is no dry-run mode.** Every call is live.
- Errors come back as `{"status": …, "error": "…", "description": "…"}`. Not RFC 9457.

## Steps

1. **Check the vocabulary.** Call `getVocabulary` to get the controlled vocabulary the create calls
   validate against. Sending a value outside it returns `400`.

2. **Create the movie root.** `createMovieRoot` (`POST /movies`). The root is the abstract title —
   the thing that is the same across languages and cuts. It returns a `gnID`. Keep it.
   Reverse with `deleteMovieRoot` (`DELETE /movies/{gnID}`).

3. **Create a version under the root.** `createMovieVersion`
   (`POST /movies/{rootGnID}/versions`), passing the root `gnID` from step 2. A version is a specific
   cut or language. It returns its own `gnID`.
   Reverse with `deleteMovieVersion`.

4. **Create the presentation.** Two paths, and they are not equivalent:
   - `createMoviePresentationWithVersion` (`POST /movies/versions/{versionGnID}/presentations`) creates
     it already attached to the version. Prefer this.
   - `createMoviePresentation` (`POST /movies/versions/presentations`) creates an unattached
     presentation, which you then attach with `mapMoviePresentationsToVersion`
     (`PATCH /movies/versions/presentationRelationships`).
   The mapping is its own resource — reverse it with `deleteMoviePresentationToVersionMapping` without
   destroying either side. Reverse the presentation itself with `deleteMoviePresentation`.

5. **Attach imagery.** `createImage` (`POST /images`) uploads the image and returns a `gnID`; then
   `mapProgramImages` (`POST /programImages`) binds it to the presentation. Images are many-to-many
   with presentations, so unbinding and deleting are separate acts: `unmapProgramImages`
   (`PATCH /programImages`) removes the link, `deleteImage` removes the image.

6. **Verify before you publish.** Call `getMoviePresentation` and confirm the record reads the way you
   expect. This is the last reversible moment.

7. **Publish.** `publishPresentation` (`POST /publish`) submits one presentation, or
   `bulkPublishPresentations` (`POST /bulk/publish`) submits many.

   > **Stop and read this.** `POST /publish` pushes the record into Gracenote-licensed datasets
   > distributed across 85+ countries. The GN IDS contract declares **no unpublish, withdraw, retract
   > or rollback operation**, and Gracenote publishes no window in which a publish can be undone.
   > Every other step in this skill has a named reversal; this one does not. Get human sign-off before
   > calling it, and never call it speculatively or as part of an automated retry.

8. **Confirm.** `getPublications` (`GET /publications`) lists what has been published.

## Batch variant

For volume, `createMovieBatch` (`POST /bulk/movies/batches`) accepts a batch and
`queryMovieBatch` (`GET /bulk/movies/batches/{gnID}`) polls its status. The same publish warning
applies to `bulkPublishPresentations`, multiplied by the batch size.

## Error handling

| Status | Meaning on this API |
|---|---|
| 400 | Invalid parameters or a value outside the controlled vocabulary |
| 401 | Missing or invalid `GN-APIKEY` |
| 403 | Your key is not entitled to this feature |
| 404 | `gnID` not found |
| 409 | Conflict — the object already exists or is in a conflicting state. **Treat this as a signal that a previous call succeeded, not as a failure to retry past.** |
| 413 | Payload too large |
| 415 | Unsupported media type |
| 422 | Well-formed but semantically rejected |
| 500 | Retry after a delay |
