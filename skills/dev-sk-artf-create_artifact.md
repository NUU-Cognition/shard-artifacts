---
description: "Create a NUU Artifact: write one self-contained HTML file, publish it with flint orbh artifacts publish, check the result, pin its URL, and record its address; with --respond for a page that takes an answer of its viewer"
---

> [!important] THIS FILE IS AN INSTRUCTION. WHEN REFERENCED IT IS MEANT TO BE TAKEN AS AN ACTION.

Run `flint shard start artf` if you haven't already.

# Skill: Create Artifact

Write one HTML page and publish it as a new NUU Artifact, so that a person can open it in the browser.

Use this skill when the work makes a page that a person must open: a report, a board, a review page, a demo, a chart, or a form that asks for an answer. When the page is a new version of an artifact that exists, use [[dev-sk-artf-use_artifact]] instead.

# Input

- The content of the page, or an HTML file that exists
- (Optional) A title and a description. The default title is the text of `<title>` of the file, else the file name.
- (Optional) The workspace: an org (`--org <slug>`) or the personal workspace (`--personal`). The default is the org of the Flint above the working folder, else the personal workspace.
- (Optional) Who can send a response from the page: `me` or `org`

# Actions

1. **Look first.** Find out if the work published this page before: the task or the report of the work, the `OBJECTS` block of your Page, or

   ```bash
   flint orbh artifacts list --mine --query "<words of the title>" --json
   ```

   When an artifact exists, stop here and use [[dev-sk-artf-use_artifact]] (`--update`).

2. **Write the HTML file.** Keep the file in the Flint when it is finished work (for example under `Media/`), or in the scratch folder of the session when it is a draft.
   - **One file, at most 10 000 000 bytes.** Put the CSS and the scripts in the file. Give each image as a `data:` URI or as an absolute `https://` URL. A relative link to another file does not work.
   - **A `<title>`.** It is the default title of the artifact.
   - **No secret.** Each reader of the artifact can read the source.
   - **The page is in a frame with no same origin.** It cannot read a cookie of NUU, and each request that it makes is a cross-site request. Scripts, forms, pop-ups, and downloads work.
   - **More than one view in one file.** Use tabs or `#` routes inside the file. A second file is not possible.
   - **A viewport `<meta>` and a layout that works at phone width.** A person can open the URL on a phone.
   - This shard does not give the style of the page. Use the design rules of the work.

3. **For a page that takes an answer, add the bridge code.** Start from the asset `Shards/Artifacts/assets/ast-artf-response_page.html` (copy it, then change the form), or read [[dev-knw-artf-bridge]] § The Page Code. The page sends one JSON value, the person confirms it in NUU Artifacts, and the answer comes to your session.
   - The page gets back only `{ ok: true }` or `{ ok: false, code }`. It gets no name and no data of the person.
   - Do not ask the person to type a secret into the page.
   - The page must say so when it has no viewer page ("Open in a new tab", a downloaded file).

4. **Check the file.** It is one `.html` file, it is not larger than the limit, and it has a `<title>`. When you can, open it in a browser and check that it shows with no other file.

5. **Publish.**

   ```bash
   flint orbh artifacts publish "<file.html>" --title "<title>" [--description "<text>"] [--org <slug> | --personal] --json
   flint orbh artifacts publish "<file.html>" --title "<title>" --respond me --json      # a page that takes the answer of the publisher
   flint orbh artifacts publish "<file.html>" --title "<title>" --respond org --json     # a page that takes the answer of each member of the org
   ```

   - `--respond me` is the default choice for a form. Use `--respond org` only when the work asks for the answers of the org. It is not possible in the personal workspace.
   - `--tools <app>.<tool>,…` lets the page ask its viewer to run a tool of a NUU app. Use it only when the operator asks for it. In production it answers `not-configured`.
   - `--no-add` publishes with no add of the object to your session.

6. **Read the JSON.** stdout holds one document: `{ artifact, version, created, added }`.
   - `artifact.address`, `artifact.url`, and `version` (1 for a new artifact). `created` is `true`.
   - `added` is `{ sessionId, itemId, already, subscribed }`: the artifact is now an object of your session. With `--respond`, `subscribed` must be `true`: a response then sends a message to your session.
   - `added` is `null` with `ORBH_SESSION_ID` set and no `--no-add`: the add failed and the publish is done. Add the object yourself: `flint orbh objects add "<address>"` (with `--subscribe` for a page that takes an answer).

7. **Check the published copy.**

   ```bash
   flint orbh artifacts get "<address>"
   ```

   The lines show the title, the workspace, the owner, the current version with its size and its SHA-256, and (with an audience) the line `Responses:`.

8. **Pin the URL**, so that the person opens it from the Orbh menu:

   ```bash
   flint helper pins add "<url>" --name "<title>"
   ```

9. **Record it and tell the person.** Write the address and the URL in the Mesh file of the work (the task note, the report, or the notepad) and in the result of the turn. For an Ops task, a short comment can give the URL: `ops comment add <task> "<title>: <url>"`. Tell the person the URL, and for a form: what you ask and who can answer.

10. **For a form, wait for the answer** with [[dev-sk-artf-use_artifact]] § Get the Answer of a Person.

When a step fails, read the error code in [[dev-knw-artf-api]] § Errors. Publish again at most one time after `hash-mismatch`, `upload-missing`, or `limit`.

# Output

- A new NUU Artifact: its address, its URL, and version 1
- The artifact as an object of the session (with a subscribe for a page that takes an answer), and its URL as a pin
- The address and the URL in the Mesh file of the work and in the result
