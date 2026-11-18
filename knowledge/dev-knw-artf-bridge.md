---
description: "The artifact bridge: how a page in the frame sends a response of its viewer, the messages and the result codes, the confirm, the page code, the page state (a page keeps its fields for each viewer), the rules of the server, the push that wakes the session, and the tools of NUU apps"
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
| `{ nuu: 1, kind: "state-save", id, form?, data }` | Keep `data` (a JSON value, at most 64 000 bytes) as the page state of the viewer. No confirm. § The Page State |
| `{ nuu: 1, kind: "state-load", id, form? }` | Get the page state back |
| `{ nuu: 1, kind: "state-clear", id, form? }` | Remove the page state |

| Answer of the viewer page | Meaning |
|---|---|
| `{ nuu: 1, kind: "ready", id, respond, tools, state: true }` | `respond` is true when this viewer can send a response. `tools` is the list of the declared tool names. `state: true`: the viewer page keeps the page state (for each viewer). |
| `{ nuu: 1, kind: "result", id, ok: true }` | The request worked |
| `{ nuu: 1, kind: "result", id, ok: false, code }` | The request did not work |
| `{ nuu: 1, kind: "state", id, data }` | The answer to `state-load`: the value, or `null` (no state) |

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

**A page gets no data of its viewer that its author did not have.** `ready` says only what the page can ask for, and `result` says only whether the request worked. The one value that comes back is the page state, and only from a page of the same publisher (§ The Page State). The reason: a page can send data to any server, so its author would see each value that the page gets.

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

## The Page State

A page in the frame has no storage of its own: its origin is opaque. Without the page state, a person who leaves the page (a reload, a closed tab, a link in NUU Artifacts, a new version) loses each mark and each field. With it, the page keeps one JSON value for each viewer and gets it back at the next load: on another day, in another browser, and in a new version of the same publisher.

- **Use it for each page that keeps the state of a form:** marks, comments, a draft answer, filters, the open tab. A page that only shows content does not need it.
- **Only the viewer reads it.** Not the owner of the artifact, not an admin, not an agent: the CLI and the object tools have no read of a state. To give an answer to the session, the page still sends a response (`respond`, with a confirm).
- **The key** is the artifact, the person, the PUBLISHER of the shown version (the person, not the session), and the form key of the page. A new version that the same person publishes (from any session) gets the state of the older version. A version that another person publishes (an editor of a share) gets nothing.
- **The form key** (`form`) is 1 to 64 characters: letters, digits, and `. _ : -`; the first one is a letter or a digit. Absent: `default`. Give a form key with a version (`review-v1`), and put a `schema` number in the value: when the form of the value changes, change the key, or read the old value with care.
- **Two places.** The viewer page writes a copy into the browser at once, and sends the newest value to the backend at most each 1.5 seconds, and at once when the page goes. So a fast close of the tab loses nothing, and the state follows the person to another browser.
- **One value at a load:** the newer of the copy and the backend. The viewer answers a load in at most 5 seconds: with no answer of the backend, the copy, else `ok: false` (never `null` for a state that did not load).
- **`ok: true` of a save** says that the copy in the browser is written.
- **Two tabs:** the last save wins. An open page gets no message when another tab saves.
- **"Open in a new tab"** has no viewer page and no state. Tell the person that the marks are not kept there.
- **Limits:** 64 000 bytes of JSON text; 20 states of one person on one artifact; 60 saves of one state in a minute; 30 new states of one person in 10 minutes. The page saves after a change (at once after a click, 300 ms after the last key), not in a loop.
- **The life:** a state stays until the page clears it, or the artifact is deleted for good (30 days after a delete).
- **The version pin.** With no `?v=` in the URL, the viewer keeps the version that it loaded. When a session publishes a new version, the person sees "v<n> is new · Load" and the page stays. The click loads the new version, which gets the state when the same person published it.

### The Helper

The asset `Shards/Artifacts/assets/ast-artf-response_page.html` has the helper `NuuState` (load, save, clear, autosave) and uses it to keep its fields. Copy the helper into each page that keeps the state of a form.

```js
const FORM_KEY = "review-v1";
const saver = NuuState.autosave(FORM_KEY, () => ({ schema: 1, marks }), (result) => {
  showStatus(result.ok ? "Saved" : "Not saved (" + result.code + ")");
});
const loaded = await NuuState.load(FORM_KEY);      // { ok: true, data } (data: the value, or null) | { ok: false, code }
if (loaded.ok && loaded.data && loaded.data.schema === 1) marks = loaded.data.marks;
// After a click: saver.now(). After a key in a text field: saver.later(). After a send: keep the state, or NuuState.clear(FORM_KEY).
```

Rules for the page:

1. Load the state one time, at the start, before the person can change a field. When the load fails (`ok: false`), do not save in that visit: the save would replace a state that did not load. Say so to the person.
2. Show the result of a save: "Saved" or "Not saved (<code>)".
3. Keep a mark of a send in the state (for example the time and the keys of the sent items), so that a reload shows what is sent.
4. Never put a secret into the state. The state is the data of the person, but the page and its publisher can read it.

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
