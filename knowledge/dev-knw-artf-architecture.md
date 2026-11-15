---
description: "The architecture of NUU Artifacts: the parts (web app, backend, NUU Storage, connector, CLI, Orbh), the path of a publish and of a read, the sandbox, who can do what, the object, the limits, and what is not possible"
---

# Knowledge: Architecture of NUU Artifacts

NUU Artifacts keeps HTML pages with versions and shows them to signed-in people. This file says which parts do the work, how the bytes move, and which rules decide who can read and change an artifact.

## The Parts

| Part | What it is | Where |
|---|---|---|
| Web app | The list and the viewer page for a person. Next.js on Vercel, with the one sign-in of NUU Account. | `https://artifacts.nuucognition.com` |
| Backend | The records, the rules, and the APIs. Convex. Tables: `artifacts` (with the versions), `uploads`, `responses`, `uses`. | Production `calm-toucan-933`, dev `zany-pony-686` |
| NUU Storage | The bytes of each version. A Cloudflare Worker in front of an R2 bucket. NUU Artifacts is the app `artifacts` of it. | `https://storage.nuucognition.com` (dev: `https://storage-dev.nuucognition.com`) |
| Connector | The NUU Connect surface of the backend (`/connect/v1`). It makes an artifact an object, with tools. | The site URL of the backend |
| CLI | `flint orbh artifacts`. It calls the HTTP API of the backend (`/api/v1`) with the sign-in of the NUU CLI. | The Flint CLI |
| Orbh | Keeps an artifact in the context of a session, and sends a message when it changes. | The orchestrator of the computer |

The code is in two repositories. `main`: `apps/nuu-artifacts` (the web app in `src/`, the backend in `convex/`), `apps/nuu-storage`, and `packages/storage`. `flint`: `apps/orbh-cli/src/artifacts-*.ts` (the CLI) and `packages/orbh/src/objects/` (the object item and the push).

```
                 publish                                      read (a person)
 agent ── CLI ──► backend (/api/v1) ──► record            browser ──► web app ──► backend
            │        │ signs one upload URL                              │          │ signs one read URL
            └────────┼──► NUU Storage (Worker + R2) ◄────────────────────┘◄─────────┘
                     │        the bytes of each version        the frame loads the page
                     └──► connector (/connect/v1) ──► the object ──► Orbh ──► the session
```

## The Path of a Publish

`flint orbh artifacts publish <file.html>` does three calls:

1. **Begin.** The CLI sends the size and the SHA-256 of the file, the title, the workspace, and the audience to the backend. The backend checks the person, reads the org roles of the person from NUU Orgs at that time, checks the limits, and answers with one signed upload URL. The URL is for this one file: it lives 15 minutes, it refuses a second write, and NUU Storage checks the SHA-256.
2. **Put.** The CLI sends the bytes straight to NUU Storage. The bytes do not go through the backend.
3. **Commit.** The backend checks that the stored object is the file that the begin named. Then it makes the artifact with version 1, or adds a version to the artifact (`--update`), and it sets the audience.

After the commit, the CLI adds the artifact to the session as an object (with a subscribe when the page takes responses).

**A folder** (`publish <folder>`): the begin sends the path, the size, and the SHA-256 of each file. The backend checks each path, sets each type from the extension, and answers one signed upload URL for each file. Each file is one object at `artifacts/<org>/<id>/v<n>/<path>`. The commit checks the size and the SHA-256 of each stored file. The version keeps its HTML page as its key, and the count and the bytes of its files.

## The Path of a Read

**A person.** The viewer page asks the backend for the artifact, and the backend checks the access of the person. Then the viewer page asks for one signed read URL of the version. The URL lives one to two hours, and the web app gets a new one before the end. The frame of the viewer page loads the HTML from NUU Storage with that URL.

**A folder version.** The read URL is `<storage>/p/<token>/<folder>/<entry>`: a FOLDER token in the PATH, for the folder of that one version. The browser resolves a relative link of the page (`media/clip.mp4`) to `<storage>/p/<token>/<folder>/media/clip.mp4`, so each link keeps the token, in the frame and in a new tab. NUU Storage refuses each key outside the folder, and each write. The responses of `/p/` have `Access-Control-Allow-Origin: *`, so the fonts, the module scripts, and the `fetch` reads of the page work from its sandbox. A page that stays open longer than the URL (one to two hours) cannot load a file late: load the page again. The details: `apps/nuu-artifacts/MEDIA.md` in the monorepo.

**An agent.** `flint orbh artifacts get` reads the facts. With `--source` or `--out`, the CLI gets a signed read URL and reads the bytes itself. It never prints the URL. The connector gives the same facts and the source as an object (`flint orbh objects get`).

## The Sandbox

The page runs in a frame that is closed against NUU:

- The frame comes from the host of NUU Storage, not from the host of the web app.
- Its sandbox is `allow-scripts allow-popups allow-popups-to-escape-sandbox allow-forms allow-modals allow-downloads`. It never has `allow-same-origin`, so the page has an opaque origin: no cookie, no local storage of NUU, and no sign-in.
- NUU Storage sends the page with a `Content-Security-Policy: sandbox` header and `nosniff`, so a direct open of the URL is closed in the same way.
- The page CAN load scripts and send requests to other servers, and it can open a link. So each value that a page gets is visible to its author. That is the reason why a page gets no data of its viewer.
- No other site can put the web app into a frame (`frame-ancestors 'none'`).

## Who Can Do What

| Action | Who |
|---|---|
| Read an artifact of an org (the facts, the source, the versions) | Each member of the org |
| Read an artifact of a personal workspace | Only that person |
| Publish a new artifact in an org | Each member of the org |
| Read an artifact through a share | Each person of a person share, and each member of the org of an org share |
| Publish a new version, rename | The access `owner`, and an editor of a share (the access `editor`) |
| Delete, share, change who can respond or the tools | The publisher of the artifact, an owner or admin of its org, and the person of a personal workspace (the access `owner`) |
| Send a response from the page | The audience of the artifact: only the publisher (`me`), or each person who can read it (`org`) |
| Read the responses | The access `owner` reads each response. Each other reader reads only the own responses. |

- For the CLI and the connector, the backend reads the org roles from NUU Orgs at the time of each call. A person who left an org loses its artifacts at the next call.
- A missing artifact and an artifact that the person cannot read give the same answer: `not-found`.
- A viewer must sign in. An artifact has no public link.
- A share gives `reader` or `editor`, never `owner`. An org share gives each member of that org at most `editor` (the cap of the role). The rule is `@nuucognition/relations` (the sharing of NUU Operations); the people and the orgs of a share come from the identity cache of the app. The details: `apps/nuu-artifacts/SHARES.md` in the monorepo.

## The Object

An artifact is an object of the Objects Protocol, so each NUU tool that knows objects can read it.

| Form | Example |
|---|---|
| Address | `@v1/<org>/artifact/<id>#artifacts` |
| Address of one version | `@v1/<org>/artifact/<id>@<n>#artifacts` |
| URI | `nuu://artifacts/artifact/<id>` |
| Web URL | `https://artifacts.nuucognition.com/<org>/<id>` (`?v=<n>` for one version) |

- The context of the object has the facts of the artifact, a guidance text, and the HTML source of the current version (cut at 40 000 characters).
- Its tools are `read_source`, `read_responses`, `publish_version`, and `rename_artifact` ([[dev-knw-artf-api]] § The Object Tools).
- Its version text changes with each new version, each rename, each change of the access of the person, and (for the access `owner`) each response. Orbh reads this text to know that the artifact changed.
- **The watch.** A session that subscribes gets one message of `orbh:objects` for each change. Orbh reads a subscribed artifact each 60 seconds. For a session of the publisher, a push makes Orbh read the artifact at once: the message comes about 3 seconds after the change ([[dev-knw-artf-bridge]] § The Push).

## The Limits

| Limit | Value |
|---|---|
| One version | One HTML file, at most 10 000 000 bytes, `text/html; charset=utf-8`; or a folder |
| A folder version | At most 200 files, 100 MB for each file, 250 MB in all; its HTML page at most 10 000 000 bytes; a path at most 200 characters; the types of the list of the server |
| Title, description | At most 200 and 2000 characters |
| Versions of one artifact | At most 200 |
| Uploads that one person starts in 24 hours | At most 300, and at most 2000 files and 2 GB |
| HTML text in an object tool (`publish_version`) | At most 2 000 000 characters |
| Source in the object context | The first 40 000 characters |
| `read_source` | 60 000 characters by default, 200 000 at most for each read |
| A list | 50 rows by default, 200 at most |
| One response | A JSON value, at most 64 000 bytes as JSON text |
| Responses of one person | At most 30 in 10 minutes |
| Responses of one artifact | At most 2000 |
| A read of the responses | At most 200 rows and 2 000 000 bytes; `next` gives the rest |
| A deleted artifact | The bytes go after 30 days |

## What Is Not Possible

- **A link outside the folder.** A folder version reads only its own files. A root link (`/x.png`), a link to another version, and a link to another artifact do not work.
- **A file of a page of one file.** A page of one file has no other files: a relative link of it fails (`missing_token`). Publish the folder.
- **Server code.** A page has only the browser. It has no backend of its own.
- **Data of the viewer in the page.** A page learns only whether its request worked ([[dev-knw-artf-bridge]]).
- **A public link.** Each viewer signs in and must be able to read the artifact.
- **An edit of a version.** A version never changes. A change is a new version.
