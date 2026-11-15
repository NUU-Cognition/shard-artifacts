---
description: "NUU Artifacts from the view of a person: the sign-in, the list, the viewer page and its controls, the versions, the responses, the confirm of a response, and what to tell a person when you give an artifact"
---

# Knowledge: The View of the Person

An agent publishes an artifact for a person. This file says what the person sees and does in the web app `https://artifacts.nuucognition.com`, so that the agent can give clear directions.

## The Sign-In

- A person signs in with the one sign-in of NUU Account. Signed out, the app shows the sign-in panel.
- A person can open an artifact of an org only as a member of that org. For each other person, the artifact does not exist.
- The URL of an artifact works on each computer and on a phone. That is the reason to give the URL, not a file path.

## The List (`/`)

- The list shows one workspace, newest first: the personal workspace, or one org of the person (`?org=<slug>`).
- "Mine" (`?mine=1`) shows only the artifacts that the person published.
- "Shared" (`?shared=1`) shows the artifacts that other people shared with the workspace: "Shared with you" in the personal workspace, "Shared with <org>" in an org.
- Each row has the title (a link to the viewer page), the owner, the workspace, the time, the current version, and "Copy address".

## The Viewer Page (`/<org>/<id>`)

One header line, then the page of the artifact in a frame that fills the rest of the height.

| Control | What it does |
|---|---|
| "Back to the list" | Goes to the list |
| The title | The name of the artifact |
| The version button | Shows the versions. A click on one shows that version (`?v=<n>` in the URL). |
| "Responses: n" | Opens the panel of the responses: each one for the access `owner`, else the own ones |
| "Copy address" | Copies the address of the object. A person can paste it to an agent. |
| "Open in a new tab" | Opens the page with no viewer page around it |
| "Download" | Saves the HTML file |
| "Share" | Opens the Share dialog: who has access and why. The access `owner` also adds people (the NUU person picker) and orgs (the org picker) as Reader or Editor, changes a permission, and removes an access. Each other reader sees only the own access. |
| "More actions" | For the publisher and an org owner or admin: "Rename" and "Delete". For an editor of a share: only "Rename". |

- The page in the frame runs scripts, forms, pop-ups, and downloads. It cannot read the sign-in of the person.
- "Open in a new tab" and a downloaded file have no viewer page. A page that sends a response cannot send there, and it must say so.

## A Response, From the View of the Person

1. The person opens the URL and fills the form of the page.
2. The person presses the send button of the page.
3. NUU Artifacts shows the dialog "Send this response?" with the exact data that goes out. The person reads it.
4. The person presses "Send". The button works after a short moment and after the person saw the end of the data. "Cancel" sends nothing.
5. The page shows that the answer is sent. The session of the agent gets it some seconds later.

The dialog is a part of NUU Artifacts, not of the page: a page cannot change what the dialog shows. The name on the response comes from the sign-in of the person.

## What to Tell the Person

When you give an artifact to a person, say:

- **The URL**, and the title of the artifact.
- **What the page is**: a report, a board, a form.
- **For a form**: what you ask, who can answer (only this person, or each member of the org), and that NUU Artifacts shows a confirm with the data before it sends.
- **After a new version**: that the same URL shows the new version, and what changed.

Do not tell a person to open an HTML file by path on another computer. Do not ask a person to type a secret into a page.
