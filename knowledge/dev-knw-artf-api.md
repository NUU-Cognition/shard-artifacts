---
description: "The APIs of NUU Artifacts: each command of flint orbh artifacts with its flags and its JSON, the reference forms, the object verbs and tools, the HTTP API /api/v1, the public functions of the web app, the error codes, and the exit codes"
---

# Knowledge: APIs of NUU Artifacts

An agent works with NUU Artifacts through the CLI `flint orbh artifacts` and through the object verbs `flint orbh objects`. This file gives each command, and the APIs below them.

## The Reference

Each command takes one reference `<id|address|uri>`:

| Form | Example |
|---|---|
| Id | `j577w7egh4y7rqwm740pz50xcx8j5cgg` |
| Address | `@v1/nuucognition/artifact/j577w7egh4y7rqwm740pz50xcx8j5cgg#artifacts` |
| Address of one version | `@v1/nuucognition/artifact/j577w7egh4y7rqwm740pz50xcx8j5cgg@2#artifacts` |
| URI | `nuu://artifacts/artifact/j577w7egh4y7rqwm740pz50xcx8j5cgg` |

A web URL is not a reference: take the id from its path. The object verbs (`flint orbh objects`) need the address or the URI. Put an address in quotes in a shell: it has `#` and `@`.

## The CLI: `flint orbh artifacts`

```bash
flint orbh artifacts publish <file.html|folder> [--entry <path>] [--title <text>] [--description <text>] [--org <slug> | --personal]
                                         [--update <reference>] [--respond me|org|off] [--tools <app>.<tool>,…|off]
                                         [--no-add] [--json]
flint orbh artifacts list [--org <slug> | --personal] [--mine | --shared [--cursor <next>]] [--query <text>] [--limit <n>] [--json]
flint orbh artifacts get <reference> [--version <n>] [--source | --out <file>] [--json]
flint orbh artifacts open <reference> [--version <n>] [--print]
flint orbh artifacts versions <reference> [--json]
flint orbh artifacts responses <reference> [--limit <n>] [--before <ms>] [--json]
flint orbh artifacts share <reference> --to org:<slug>|person:<user id>|@<handle> [--permission reader|editor] [--json]
flint orbh artifacts shares <reference> [--json]
flint orbh artifacts unshare <reference> <share id|org:<slug>|person:<user id>|@<handle>> [--json]
flint orbh artifacts delete <reference> [--yes] [--json]
```

| Command | What it does | Notes |
|---|---|---|
| `publish` | Publishes an HTML file or a folder as a new artifact, or as a new version (`--update`) | The file has at most 10 000 000 bytes. A folder: its HTML page (`index.html`, or `--entry <path>`) and its files, at most 200 files, 100 MB for each file, 250 MB in all; hidden files are skipped, and a symbolic link is refused. The default title is the text of `<title>`, else the file or folder name. The default workspace is the org of the Flint above the working folder, else the personal workspace. With `ORBH_SESSION_ID`, it adds the object to the session; `--no-add` stops that. |
| `list` | Lists the artifacts that you can read, newest first | `--mine`: only the artifacts that you published first. `--shared`: the artifacts that other people shared with you (with `--org`: with that org), newest share first. One read looks at a part of the share rows: when more follow, the output ends with the next command (`--cursor <next>`, with the same filters). `--query`: the title must match. `--limit`: 1 to 200, default 50. |
| `get` | Prints the facts, or the HTML | `--source`: the HTML on stdout. `--out <file>`: the HTML into a file. `--version <n>`: one version (default: the current version, or the `@<n>` of the address). |
| `open` | Opens the web page in the browser | With no terminal, and with `--print`, it only prints the URL. |
| `versions` | Lists the versions, newest first | Text: the number, the size (a folder version: `<n> files, <size>`), the date, the creator, the current mark. `--json` also has the SHA-256, and `files` and `bytes` for a folder version. |
| `responses` | Prints the responses that people sent from the page, newest first | `--limit`: 1 to 200, default 50. `--before <ms>`: only older responses (the `next` of the read before). |
| `share` | Shares an artifact with an org or one person, as `reader` (the default) or `editor` | Only the access `owner`. A second share to the same subject changes its permission. `--json`: `{ shareId, created, changed, permission }`. |
| `shares` | Prints who has access: the org of the artifact, the publisher, and each share with its id | The access `owner` sees each line. Another reader sees only the own lines. `--json`: `{ artifact, title, canManage, summary, lines }`. |
| `unshare` | Removes one share | The argument is the share id, or the subject of the share (`org:<slug>`, `person:<user id>`, `@<handle>`). The access ends at the next call. |
| `delete` | Deletes an artifact (a soft delete; the bytes go after 30 days) | In a terminal it asks. With no terminal, `--yes` is required. |

**`--respond`.** `me` (the same as `owner`): only the publisher can send a response. `org`: each person who can read it: each member of the org of the artifact, and each person of a share (not in the personal workspace). Only the access `owner` changes it. `off`: nobody. A publish with `--respond` subscribes the session to the artifact. An update with no `--respond` keeps the audience.

**`--tools`.** A comma list of `<app>.<tool>`, or `off`. The page can then ask its viewer to run those tools. It works only when the deployment has the tools of the bridge on (production: off). Use it only when the operator asks for it.

### The JSON of each command

| Command | `--json` prints |
|---|---|
| `publish` | `{ artifact, version, created, added }`. `created` is true for a new artifact. `added` is `{ sessionId, itemId, already, subscribed }`, or `null`. |
| `list` | `{ items }` |
| `get` | `{ artifact }` (with `--version`: also `version`; with `--source`: also `source`; with `--out`: also `out`) |
| `versions` | `{ id, address, current, versions }` |
| `responses` | `{ id, address, title, access, total, items, complete, next? }`. Each item: `{ id, version, by, name?, data, createdAt, mine }`. `data` is a JSON text. |
| `delete` | `{ ok, id }` |

The fields of `artifact`: `id`, `org`, `workspace` (`org` or `personal`), `title`, `description?`, `owner`, `ownerName?`, `source?` (the session that published it), `current`, `versions` (each one `{ n, size, sha256, createdAt, createdBy, actor? }`: the number, the size, the SHA-256, the time, the person, and the session), `createdAt`, `updatedAt`, `address`, `uri`, `url`, `access` (`owner` or `member`), `bridge?` (the audience and the tools), `canRespond?`, and `responses?` (`{ count, lastAt }`, only for the access `owner`).

### Text output of `publish`

```
Published <title> (v1)            # or: Updated <title> (v<n>)
Address: @v1/<org>/artifact/<id>#artifacts
URL: https://artifacts.nuucognition.com/<org>/<id>
Responses: <who can send a response from the page>     # only with an audience
Object: added to session <id>                          # or: Object: session <id> holds it already (<item>)
Subscribed: a response sends a message to the session. # only with an audience
Read the responses: flint orbh artifacts responses '<address>'
```

A warning of a failed add of the object is on stderr: `Warning: the object was not added to session <id>: <reason>`. The publish is done then.

### The sign-in, the site, and the mode

- The sign-in is the sign-in of the NUU CLI: `nuu login`.
- The site URL of the backend comes from the connect registry. The env `NUU_CONNECT_ARTIFACTS_URL` wins.
- The data mode is production. `NUU_OBJECTS_MODE=dev` selects the dev deployment.

## The Object Verbs: `flint orbh objects`

```bash
flint orbh objects get "<address>"                       # read one time; it writes nothing
flint orbh objects add "<address>" [--subscribe]         # keep it in the session (a link); --subscribe: a message when it changes
flint orbh objects list                                  # the items of the session
flint orbh objects show <item id> [--full]               # read an item again
flint orbh objects remove <item id>                      # remove an item
flint orbh objects tools "<address>"                     # list the tools
flint orbh objects use "<address>" <tool> [flags]        # run one tool; <tool> --help prints its flags
```

### The Object Tools

| Tool | Kind | Flags | Notes |
|---|---|---|---|
| `read_source` | read | `--offset <n>` `--limit <n>`; the version with `--input '{"version": <n>}'` | The HTML source in parts: 60 000 characters by default, 200 000 at most. `complete: false`: read again with `offset` = offset + length. |
| `read_responses` | read | `--limit <n>` `--before <ms>` | The same read as `flint orbh artifacts responses`. |
| `publish_version` | mutates | `--html <text>` or `--html-file <path>`; `--title`, `--description`, `--respond owner\|org\|off` | A new version from the full HTML text, at most 2 000 000 characters. Only with the access `owner`. For a larger file, use the CLI `publish --update`. It makes a version of one file: of a folder version, publish the folder with the CLI, else the media links of the page fail. |
| `rename_artifact` | mutates | `--title <text>` `--description <text>` | 1 to 200 characters for the title, at most 2000 for the description. An empty description removes it. Only with the access `owner`. |

The connector also has the tool `publish_artifact` (a new artifact from an HTML text of at most 2 000 000 characters, with `title` and an optional `org`). It is for an agent in another NUU app. In a Flint session, use the CLI `publish`.

The exit codes of `flint orbh objects use`: 0 ok; 1 a refusal of a flag or a name; 2 `bad-address` or `no-adapter`; 3 `not-configured`; 4 `unreachable`; 5 `not-found`; 6 `denied`; 7 `bad-response`; 8 `tool-failed`.

## The HTTP API: `/api/v1`

The CLI calls this API. An agent uses the CLI, not the API. The facts are here for work on the tools.

- Each call is `POST <site URL>/api/v1/<route>` with a JSON body of at most 200 000 characters.
- The person is the NUU CLI access token of the request (`Authorization: Bearer`). The token proves only who the person is. The backend reads the org roles from NUU Orgs at the time of the call.
- `X-NUU-Actor` names the session of the call (attribution only).
- A refusal is a status that is not 2xx, with `{ error: { code, message } }`.

| Route | Body | Answer |
|---|---|---|
| `/publish/begin` | `size`, `sha256`, `title?`, `description?`, `org?`, `update?`, `respond?`, `tools?` | `{ uploadId, artifactId, version, upload: { url, method: "PUT", headers, maxBytes, expiresAt } }` |
| `/publish/begin` (a folder) | `files: [{ path, size, sha256 }]`, `entry?` (default `index.html`), and the same other fields | `{ uploadId, artifactId, version, entry, files: [{ path, url, method: "PUT", headers, maxBytes }], expiresAt }`. The server sets the `Content-Type` of each file from its extension. |
| `/publish/commit` | `uploadId` | `{ artifact, version, created }` |
| `/list` | `org?`, `query?`, `mine?`, `shared?`, `cursor?`, `limit?` | `{ items }`; a shared list also has `complete` and `next` (the `cursor` of the next read) |
| `/get` | `artifact` | `{ artifact }` |
| `/read-url` | `artifact`, `version?`, `download?` | `{ url, version, size, sha256, expiresAt }`; a folder version also has `files` and `bytes`, and its `url` is the URL of its HTML page with a folder token (`/p/<token>/<key>`). `download` gives the HTML page only. |
| `/rename` | `artifact`, `title?`, `description?` | `{ artifact }` |
| `/delete` | `artifact` | `{ ok, id }` |
| `/responses` | `artifact`, `limit?`, `before?` | `{ id, address, title, access, total, items, complete, next? }` |
| `/shares/list` | `artifact` | `{ artifact: { id, address, access, via }, title, canManage, summary, lines }` |
| `/shares/add` | `artifact`, `to` (`org:<slug>` or `person:<user id or @handle>`), `permission?` | `{ shareId, created, changed, permission }` |
| `/shares/permission` | `shareId`, `permission`, `artifact?` | `{ shareId, created, changed, permission }` |
| `/shares/remove` | `shareId`, `artifact?` | `{ ok, shareId }`. With `artifact`, a share of another artifact is `not-found`. |

`upload.url` and the `url` of `/read-url` are signed URLs of NUU Storage. Never print one, and never write one into a note.

## The Functions of the Web App

The web app calls public functions of the backend with the sign-in of the browser: `artifacts:workspaces`, `list`, `get`, `readUrl`, `rename`, `remove`, `bridge:respond`, `responses`, `use`, `state:load`, `save`, `clear` (the page state: [[dev-knw-artf-bridge]] § The Page State), and `shares:forEntity`, `viewerAccess`, `share`, `setPermission`, `unshare`. These functions refuse the token of the NUU CLI, so no CLI command and no object tool reads a page state. The Share dialog also reads the people search and the org search of the identity cache (`identityCache:searchPeople`, `searchOrgs`). The one function that the NUU CLI token can call is `objectsPush:version`: it gives only a fingerprint of an artifact, and only to its publisher ([[dev-knw-artf-bridge]] § The Push).

## Errors

| Code | Status | Meaning and what to do |
|---|---|---|
| `unauthenticated` | 401 | No sign-in. Ask the person to run `nuu login`. A headless session cannot sign in: report it. |
| `bad-request` | 400 | A wrong field. With `--respond org`: the personal workspace has no org (use `--respond me`, or `--org <slug>`). With `--tools`: the app has no such tool, or the tool only reads. |
| `forbidden` | 403 | You cannot do this: a change of an artifact with the access `member`, a share, a delete, or a change of `--respond` with the access `editor`, or an org where you have no role. Ask the publisher, or create a new artifact. |
| `self-target`, `same-org`, `personal-target` | 400 | A share to yourself, to the org of the artifact (its members can read it already), or to a personal workspace as an org. Share with another person or another org. |
| `person-unknown`, `org-unknown` | 404 | NUU does not know the person (or the handle names no one person) or the org. Check the handle or the slug. |
| `not-found` | 404 | The artifact does not exist, or you cannot read it. Check the reference and the org. |
| `too-large` | 413 | The file is larger than 10 000 000 bytes, a folder is larger than its limits (200 files, 100 MB for each file, 250 MB in all), or a text is larger than its limit. Make it smaller. |
| `hash-mismatch`, `upload-missing` | 409 | The upload did not arrive whole (of a folder: the message names the first files that are missing). Publish again one time. |
| `limit` | 429 | Too many requests or too many versions, or more than 300 uploads, 2000 files, or 2 GB of one person in 24 hours. Wait, then try again one time. |
| `not-configured` | 503 | The deployment has no storage, or the tools of the bridge are off. Report it with the command. |
| `tool-failed` | 502 | A tool of a NUU app refused or failed (the tools of the bridge). |
| `outcome-unknown` | 504 | A tool of a NUU app gave no clear answer: it can have run. Look in the app before a second try. |
| `internal` | 500 | A fault of the server. Report it with the command. |

The exit codes of `flint orbh artifacts`: 0 ok; 1 a refusal of the server or a failed request; 2 a wrong command line.
