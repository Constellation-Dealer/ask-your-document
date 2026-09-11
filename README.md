# Ask Your Document — Gateway Chat with Agentic Loop

A hands-on exercise where you upload a PDF, wait for it to be indexed, and then ask questions about it through the TargetMCP Gateway's streaming Chat API. The Gateway's LLM agent calls the right tools automatically — you watch the agentic loop happen in real time via SSE events.

## Getting Started

1. Clone this repo
2. Copy `.env.example` to `.env` and fill in your credentials (from the Champion Portal → Workshop Details)
3. Make sure Node.js LTS is installed
4. Run `npx --yes http-server . -a localhost -p 3000 -c-1`
5. Open `http://localhost:3000`

Then open <http://localhost:3000>.

If 3000 is taken it is almost certainly the previous workshop's server — stop that
and start this one again. `5173` also works. Those two are the only ports IDMS and
the Gateway accept, and `127.0.0.1` is a different origin that they do not.


6. Click **Run the loop** to see Step 1 execute (and a prompt to implement the rest)

> **Do NOT open `index.html` directly as a file** (`file://...`). A `file://` page has no HTTP
> origin at all, so every API call is refused — this is about serving over HTTP, not about which
> port you pick. Both `http://localhost:3000` and `http://localhost:5173` are allowlisted.

## The PDF you bring

### 🔴 One corpus, everyone's documents

**Everyone in this workshop signs in as the same dealer, and every upload lands in the same shared
corpus.** There is no per-person partition in this exercise. Whatever you upload can be retrieved
and quoted back, word for word, in the answer to somebody else's question — and theirs in yours.

So: **do not upload anything you would not hand round the room.** No customer data, no contracts,
no internal financials, nothing under NDA.

What works well instead: an **equipment manual**, a **spec sheet**, or any **public PDF** you like.
The exercise only needs a document with real text in it, and one you know well enough to tell a
good answer from a plausible one.

Naming the file distinctively — `compact-tractor-manual-yourname.pdf` — makes it easier to ask a
question you know only your document answers. That is a **retrieval aid, not a control**: it helps
you steer your own question, it keeps nobody else out. Your upload now carries a tag with your name
on it as well — see *Your own corner of the corpus* below — which is a better retrieval aid and
still not a control.

## Your own corner of the corpus

The warning above stays true: one dealer, one corpus, nothing you upload is private to you. This is
the other half of it. Within a shared corpus, your upload can still say which of us put it there.

TargetUMH lets a file be **tagged with an entity** — a type and an id, stored next to the file — and
the retrieval tool can filter on it. That is not a workshop invention. Of this dealer's roughly
3,400 documents, about 2,400 already carry such a tag, and the platform's OEM manual lookup filters
on exactly `entityType="Model"`. This exercise was one of the few things not using it, so every
question you asked searched all 3,400 documents at once.

Now it uses it. Two things changed, and they are **not the same kind of thing**.

**On upload — enforced.** `uploadPdf` now sends `entityType: "Model"` and `entityId: <your handle>`
along with the file. TargetUMH records the tag, and rejects the upload with a 400 if only one of the
pair arrives — so they always travel together. Your handle comes from `participantEntityId()` in
`helpers.js`: the local part of the IDMS username already in your `.env`, lowercased and normalised,
so `first.last@…` becomes `first-last`. Nothing new to configure and no extra credential. It is the
same every run, which is what lets today's run find what you uploaded yesterday. The connection
panel at the top of the page shows the tag you will get.

**On the question — steered, not enforced.** The same entity goes to the Gateway with your question.
But the Gateway's *agent* chooses the tool arguments, and it is free to leave yours out. The entity
is a request, not a filter. Retrieval usually narrows to your own document. It is not guaranteed to.

### The observable

So how do you know which of those happened? Not from the answer — a good answer looks the same
either way.

The trace now carries a **Retrieval scope** card under the Gateway step, and it prints the arguments
the agent *actually* passed to the media tools.

Green means every search of the corpus it ran carried your `entityType`/`entityId` — or that it did
not need to search at all, because it went straight to your document by its id, which is narrower
still.

Red means at least one search did not carry it, and there are two ways to be red. Either none of
them did, or, the one worth watching for, **some did and some did not**. A turn is not one search:
the agent can run several and it does not have to treat them alike. The card calls that case
**MIXED**, and then names the searches that went out wide, because those are the ones that read
everybody's documents.

A red card is not a bug in your loop. It is what "steered, not enforced" looks like from outside.

There is a third thing the card can be, and it is the interesting one: **neither**.

What it judges is narrower than "every tool the agent called". Only calls it **recognises as
retrieving document content** count towards the verdict. The rest split in two, and the split
matters:

- **Calls it knows are not retrieval.** Generating embeddings is processing; listing the entity
  *names* in use is not reading documents; a tool the Gateway ran on a different server never
  touched this corpus. These appear under *not counted either way* and take nothing away from a
  green card.
- **Calls it does not recognise at all.** A tool name this page has never seen, running where the
  media tools run. It *might* have searched wide — there is no way to tell from here. So it does not
  turn the card red, but it does take away the card's right to say **every**. The headline changes
  to *"the searches this page can account for carried your entity"*, and the call is named under
  **NOT RECOGNISED**.

That is the honest reading: the searches I can see carried your entity, and there was also a call I
cannot vouch for. A universal claim about a turn containing retrieval nobody classified is a claim
the card has not earned.

It cuts both ways on purpose. A green card on a run that went wide would teach you the entity is
enforced. A red card on a run that was properly scoped would teach you it never works — and you
would have no way to tell the card was wrong. The rule it applies is written down in one place, in
`helpers.js`: look for `RETRIEVAL_TOOLS`.

Case does not count against you either way. TargetUMH matches an entity type and id without regard
to case, so if the agent writes back `model` or `First-Last`, that is still your document and the
card still counts it as scoped.

That card is the only place those arguments show up. The `tool_start` event the tool card is drawn
from carries the tool's *name* and a description and nothing else; the arguments do not reach the
page until the turn is over, on the `complete` event.

Worth doing once: ask something only your document answers and read the card. Then ask something
only somebody else's document could answer, and read it again.

### Still not a privacy control

Tagging changes what gets **retrieved**. It does not change what can be **reached**. Your document
is still in the shared corpus, still readable by everyone in the cohort, and a question asked
without the scope — or an agent that ignores it — will still find it. **The warning above is the one
that governs what you upload.**

## File Structure

```
ask-your-document/
├── .env.example    ← Copy to .env and fill in credentials
├── index.html      ← Page structure (no need to modify)
├── styles.css      ← UI styling (no need to modify)
├── helpers.js      ← Auth, API helpers, UI wiring (no need to modify)
└── loop.js         ← YOUR CODE GOES HERE (the agentic loop)
```

**You only need to edit `loop.js`.**

## What You Write

Open `loop.js` and look for the `TODO: YOUR CODE HERE` comments. Step 1 (Upload) is done for you as an example. You fill in Steps 2-4 (~20 lines total):

1. **Polling loop** — check ingestion status, wait, repeat until embeddings are ready.
   Wait only while `isIngestionInFlight(...)` is true; every other status is terminal.
   Bring a PDF whose text you can select in a reader — TargetUMH answers `Skipped` for a
   scan with no text layer, and there is then nothing for the agent to retrieve.
2. **Ask the Gateway** — send your question to `chatWithGateway()` with SSE callbacks that render each tool call the LLM agent makes
3. **Show the answer** — display the Gateway's composed response

The key difference from a traditional RAG loop: you do NOT call `vector_search_media` or `get_document_chunks` yourself. The Gateway's LLM agent decides which tools to call and calls them for you. You just watch the SSE events stream in.

## The skill and the agent behind these answers

Every answer passes through **`workshop-1-answer-style`** — a short markdown file stored on the
platform and prepended to the model's instructions. It asks for a `Source:` line naming the document the answer came from.

It reaches you through an **agent**. `helpers.js` sends `agentId: "workshop-1-agent"`, and
that agent binds this skill *by name*. That is why you get this workshop's style
and not another's: the platform picks the skill by name, not by guessing from
your question.

The agent also declares this exercise's retrieval tools; without that the agent's
tool filter fails closed and the document search returns nothing.

> 🔴 **One copy, shared by everyone in this workshop — and it cuts both ways.**
> Editing it changes the answers every other participant gets, immediately. And
> theirs changes yours: there is no per-person copy and no reset, so whoever
> published last is the version in force. An edit you made ten minutes ago may
> already have been replaced without anyone telling you — if your answers
> suddenly change shape mid-exercise, this is usually why, not the model being
> flaky. Version history is kept, so a bad edit is recoverable; the recovery is
> shared too.
>
> This session is pre-work, done alone and spread over several days, so
> "somebody publishes over you" is not hypothetical here — it is the expected
> case. Reading it teaches the same thing and leaves the next person's run
> intact.

To read or edit it: <https://dev-dealeriq.csidealer.com> → sign in with the email
and password you were sent → **Studio** → **Skills** → `workshop-1-answer-style`.
Direct link: <https://dev-dealeriq.csidealer.com/skills>. Edit, save, then
**publish** — an unpublished edit changes nothing. Re-run and the answers change
shape.

## Configuration

All configuration is in `.env`. See `.env.example` for the full list of values including API endpoints, credentials, and dealer context.

## Local Serving

This app should be served from `http://localhost:3000`.

That port is on the DEV CORS allowlist for **all three** services this page calls — IDMS for the
token, the Gateway for the chat, and TargetUMH for the upload and the ingestion poll. The third one
is easy to forget: a port allowlisted on IDMS and the Gateway but not on UMH lets you sign in and
then fails at upload. `http://localhost:5173` is the only other origin all three accept.


That matches the DEV CORS allowlist and gives the browser a real HTTP origin for loading `.env`, JavaScript, and CSS. Opening `index.html` directly via `file://` is not the intended setup.

Use this command:

```bash
npx --yes http-server . -a localhost -p 3000 -c-1
```

## Node.js Setup

If Node.js is not already installed, install the current LTS release from [nodejs.org](https://nodejs.org/en/download).

If you are using a coding agent, the easiest path is to ask it to do the setup for you. Example prompt:

```text
Install Node.js LTS if it is not already installed, then start this app on http://localhost:3000 using:
npx --yes http-server . -a localhost -p 3000 -c-1
After that, verify the page loads successfully.
```
