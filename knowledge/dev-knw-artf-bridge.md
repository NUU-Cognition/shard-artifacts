---
description: "The artifact bridge: how a page in the frame sends a response of its viewer, the messages and the result codes, the confirm, the page code, the rules of the server, the push that wakes the session, and the tools of NUU apps"
---

# Knowledge: The Artifact Bridge

The page of an artifact has no sign-in. The bridge lets the page ask the viewer page of NUU Artifacts to act for the signed-in person. The viewer page shows a confirm with the real data, and only then calls the backend with the sign-in of the person. The full design and the threats are in `apps/nuu-artifacts/BRIDGE.md` of the repository `main`.

## The Loop

```
 session ── publish --respond me ──► artifact (the session subscribes)
                                         │
 person opens the URL ──► viewer page ──► page in the frame
                              ▲               │ parent.postMessage({ nuu: 1, kind: "respond", data })
                              │ confirm        ▼
                              └── "Send this response?" ── the person confirms the exact data
                                         │
                                  backend stores the response with the person of the sign-in
                                         │ push (about 1 second)
                                  Orbh reads the artifact ──► message of orbh:objects ──► session
                                         │
 session ── flint orbh artifacts responses ──► the answer
```

## The Messages

The page sends a message to its parent window: `window.parent.postMessage(message, "*")`.

| Message of the page | Meaning |
|---|---|
| `{ nuu: 1, kind: "hello", id }` | What can this page ask for? |
| `{ nuu: 1, kind: "respond", id, data }` | Send `data` (a JSON value, at most 64 000 bytes as JSON text) as the response of the viewer |
| `{ nuu: 1, kind: "use", id, tool, input }` | Run the tool `<app>.<tool>` with `input` (a JSON object) as the viewer |

| Answer of the viewer page | Meaning |
|---|---|
| `{ nuu: 1, kind: "ready", id, respond, tools }` | `respond` is true when this viewer can send a response. `tools` is the list of the declared tool names. |
| `{ nuu: 1, kind: "result", id, ok: true }` | The request worked |
| `{ nuu: 1, kind: "result", id, ok: false, code }` | The request did not work |

`id` is a text of at most 64 characters, or a number. The answer has the same `id`. A message with no `nuu: 1` is not a message of the bridge.

| Code | Meaning | What the page does |
|---|---|---|
| `cancelled` | The person said no in the confirm | Keep the form, so that the person can send again |
| `busy` | Another request waits for the person | Tell the person to answer the open dialog |
| `blocked` | Three cancels in a row | Tell the person to load the page again |
| `forbidden` | This viewer is not in the audience, or the tool is not declared | Say that the page does not take an answer from this person |
| `limit` | Too many requests in a short time | Tell the person to try again later |
| `too-large` | The data is larger than 64 000 bytes | Send less data |
| `bad-request` | A wrong form of the message | Correct the page |
| `not-found` | The artifact is gone | Say so |
| `off` | The tools of the bridge are off | Say so |
| `unknown` | A tool gave no clear answer: it can have run | The viewer page told the person to look in the app |
| `failed` | Another failure | Tell the person to try again |

**A page gets no data of its viewer.** `ready` says only what the page can ask for, and `result` says only whether the request worked. The reason: a page can send data to any server, so its author would see each value that the page gets.

## The Confirm

The viewer page draws the confirm, not the page. The page cannot draw over it and cannot click it.

- The dialog shows the JSON text that goes out, with each hidden character made visible.
- The focus starts on "Cancel". The button "Send" works only 800 ms after the dialog opens, and only after the person saw the end of the data (a scroll to the end for long data).
- One request at a time. Three cancels in a row block the page until a load.

## The Page Code

The asset `Shards/Artifacts/assets/ast-artf-response_page.html` is a full page with this code and a small form.

```html
<script>
  const waiting = new Map();
  let count = 0;
  function ask(kind, fields) {
    const id = "m" + ++count;
    return new Promise((resolve) => {
      waiting.set(id, resolve);
      window.parent.postMessage({ nuu: 1, kind, id, ...fields }, "*");
    });
  }
  window.addEventListener("message", (event) => {
    if (event.source !== window.parent || !event.data || event.data.nuu !== 1) return;
    const resolve = waiting.get(event.data.id);
    if (resolve) { waiting.delete(event.data.id); resolve(event.data); }
  });
  async function start() {
    // "Open in a new tab" and a downloaded file have no viewer page.
    if (window.parent === window) return showNote("Open this page in NUU Artifacts to send your answer.");
    const ready = await ask("hello", {});
    if (!ready.respond) return showNote("This page does not take an answer from you.");
    // On submit:
    //   const result = await ask("respond", { data: { choice: "b", note: "…" } });
    //   result.ok is true, or result.code says why not.
  }
</script>
```

Rules for the page:

1. Check `window.parent === window` first, and say so when the page has no viewer page.
2. Add a timer to the `hello`: when no answer comes in some seconds, tell the person to open the page in NUU Artifacts.
3. Send one JSON value with a clear form, for example `{ form: "<name>-v1", choice, note }`. The name of the form lets the session read answers of an old version correctly.
4. After `ok: true`, say that the answer is sent. After `cancelled`, keep the form.
5. Do not ask for a secret. Do not send more data than the question needs.

## The Rules of the Server

The server checks each response. The viewer page is not the guard.

- **The person** is the subject of the sign-in of the browser. The page gives no name: the server writes the name of the sign-in (`by`, `name`).
- **The audience.** `me` (stored as `owner`): only the person who published the artifact first, not an admin of the org. `org`: each person who can read an artifact of an org. A personal artifact takes a response only from its person.
- **The limits.** At most 64 000 bytes for each response, 30 responses of one person in 10 minutes, and 2000 responses of one artifact.
- **The read.** A person with the access `owner` reads each response. Each other reader reads only the own responses.
- **The audience changes only at a publish.** An update with no `--respond` keeps it. A late commit of an older upload cannot give back an audience that a newer version removed.

## The Push

A response must wake the session fast.

- For each artifact that a session subscribes to, the orchestrator of Orbh follows a live query of the backend (`objectsPush:version`). The query gives a fingerprint of the artifact, and a new fingerprint makes Orbh read the artifact at once.
- **Only a session of the publisher has the push.** The query gives a fingerprint only to the person who published the artifact first. It reads no org role. A session of another person keeps the timed read (each 60 seconds).
- Measured on production (2026-11-13): the message of `orbh:objects` reached the session about 3 seconds after the response, also when the session was idle.
- The fingerprint has a key of the server. It says only that the artifact changed, never what changed.
- **An own change sends a message too.** A new version or a rename by the session itself changes the artifact, so the session that subscribes gets a message of `orbh:objects` for its own change. Only the count of the responses says that a person answered.
- **A read by the session takes the change.** When the session reads the item itself (`flint orbh objects show`, `flint orbh page`) before Orbh does, Orbh records the new version and sends no message for it. The response is still there: read it with `flint orbh artifacts responses`.

## The Tools of NUU Apps

A page can also ask its viewer to run one tool of a NUU app (`kind: "use"`), for example to create an Ops task.

- The artifact declares the tools at its publish (`--tools <app>.<tool>,…`). Only a tool that changes data can be declared: the answer of a read tool has no place to go.
- The viewer confirms each call. The page gets only `ok` or a code.
- The deployment must name the app (the env `NUU_ARTIFACTS_BRIDGE_APPS` of the backend). **In production the tools are off**: a publish with `--tools` answers `not-configured`. The operator turns them on.
- Use `--tools` only when the operator asks for it.
