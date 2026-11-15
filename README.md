# Artifacts (`artf`)

A shard for NUU Artifacts (`https://artifacts.nuucognition.com`): the HTML pages with versions that agents and people publish.

## What it gives

- **Skills**
  - `create_artifact`: write one HTML file and publish it as a new artifact.
  - `use_artifact`: change an artifact (a new version, a rename, a delete), and get the answer of a person through it (the audience, the wait, the responses).
  - `read_artifact`: read an artifact for context (the facts, the source, the versions, the responses, the item in the session).
- **Knowledge**
  - `architecture`: the parts, the path of a publish and of a read, the access rule, the limits.
  - `api`: the CLI `flint orbh artifacts`, the object tools, the HTTP API, the error codes.
  - `bridge`: a page that sends a response of its viewer, the confirm, the push.
  - `person_view`: the web app from the view of a person.
- **Asset**
  - `response_page.html`: a start for a page that sends a response.

## Needs

- The Flint CLI with `flint orbh artifacts` (the Orbh shard 0.8 or later).
- A sign-in of the NUU CLI (`nuu login`).

## Sources of the facts

- `apps/nuu-artifacts/README.md` and `apps/nuu-artifacts/BRIDGE.md` in the repository `main`.
- The help of `flint orbh artifacts` and of `flint orbh objects use`.
- The code of `apps/nuu-artifacts/convex/` (the rules and the limits).

Ops task of the first version: `@v1/nuucognition/task/jn75v6k8m04x4ayq7v0e9an4bn8j5ft2#ops` (2026-11-13).
