# I Built an AI Business Analyst for Hotel Management Using MCP + RAG, Pair-Programmed with AI

> My journey building an MCP-RAG server that lets a hotel manager ask "why did Deluxe room
> revenue drop this summer?" and get an answer that cites both the booking numbers and the
> meeting notes that explain them — and what I learned about working with an AI coding agent
> along the way.

## TLDR
1. I built a RAG system that answers hotel management questions by combining **quantitative**
   booking data (Google Sheets) with **qualitative** context from meeting notes (Google Docs) —
   exposed both as a chat UI and an MCP server.
2. I built almost all of it pair-programming with an AI coding agent (Claude Code), which was a
   genuinely fast way to go from spec to a working, deployed app.
3. It was not "describe app, get app." The parts that mattered most — reviewing what the AI
   actually wrote, understanding the architecture well enough to direct it, and the DevOps work
   to actually ship it — were still on me.

## What is it?
A hotel manager asks something like *"Deluxe room revenue is lower than expected this past
summer, why?"* The system:
1. Queries the booking data to quantify what happened (which room type, which period, how much).
2. Searches internal meeting notes for the qualitative reason (a maintenance issue, a pricing
   change, a marketing decision).
3. Synthesizes both into one answer, citing the actual numbers and the specific meeting that
   explains them.

It's built as an **MCP (Model Context Protocol) server** — so the same tools it uses internally
are also available to any other MCP-compatible client, not just the bundled chat UI — with a
**RAG (Retrieval-Augmented Generation)** pipeline over the meeting notes.

## The stack, and why

| Piece | Choice | Why |
|---|---|---|
| Agent backend | Python, FastAPI, LangGraph | LangGraph models the "check the numbers, then check the notes, then synthesize" flow as an explicit agent loop rather than one opaque prompt |
| LLM + embeddings | Google Gemini | Good tool-calling support, cheap embeddings, one provider for both chat and RAG |
| MCP server | Official `mcp` Python SDK | The point of the exercise — expose the tools over a standard protocol, not just a bespoke API |
| Vector DB | Qdrant | Lightweight, runs fine in a small Docker container at this data scale |
| Data source | Google Drive + Sheets API | Meeting notes and bookings already live there for a real hotel team — no separate data entry system to adopt |
| Chat UI | Next.js (Bun), TypeScript | Thin client — all the agent logic stays server-side, the UI just renders |
| Packaging | Docker (multi-stage builds) | Same artifact runs locally and in the cloud |
| Infrastructure | Terraform, AWS Lightsail, ECR | Declarative infra with a clean `terraform destroy`, images built once and pulled, not rebuilt on the target box |

## The journey (the honest version)

The spec-to-first-working-version part was genuinely fast — describe the scenarios, get a
working LangGraph agent with real tool calls, wired to real booking data, in one sitting.

The parts that took real time were the parts you'd expect from any software project, AI-assisted
or not:

- **The deployment target changed four times.** Hostinger → AWS Lightsail → Hostinger → AWS
  Lightsail, as real constraints surfaced (an existing box already at 60% memory, a
  misunderstanding about how Lightsail billing actually works — stopping an instance does
  *not* pause billing, only deleting it does).
- **A version pin bug that looked like something else entirely.** `"typescript": "latest"`
  quietly resolved to a new major version that restructured its package exports and broke
  Next.js's internal build tooling. The symptom (a crash trying to auto-install via a package
  manager that wasn't even in the Docker image) looked exactly like an unrelated, real Docker
  layering issue I was *also* debugging at the same time. Untangling "which bug is this
  actually" took longer than fixing either bug once identified.
- **A Docker Compose override that silently did nothing.** `build: null` looks like it should
  unset a service's build step so it pulls a pre-built image instead — it doesn't. The fix
  (`build: !reset null`) only got found by actually inspecting `docker compose config` output
  rather than trusting that the override "should" work.
- **A Terraform template that scanned its own comments.** `templatefile()` treats every `${...}`
  in the source file as an expression to evaluate — including inside my own code comments
  explaining what the placeholders do. Escaping got weird before it got right.

None of these are AI-specific failure modes. They're the normal texture of shipping software.
What changed is *how fast* I could iterate through them once each was correctly diagnosed.

## What the developer still needed to do

This is the part I think matters most for anyone assuming AI coding agents make the underlying
skills less relevant. In this project, they didn't — they just moved where the skill was applied.

1. **Review the code the AI generated.** Every bug above shipped, got tested, and got caught
   because I actually looked at what was generated and asked "is this really doing what I think
   it's doing" — not because the agent flagged its own mistake. An AI coding agent will confidently
   write a Docker override that silently no-ops, or pin a dependency to a version that breaks a
   build tool. Catching that is still a human review job.
2. **Understand the software architecture.** Deciding that the MCP server and the chat API
   should share one implementation of each tool (not two copies that could drift), that the
   vector index should be safely re-buildable from source-of-truth data rather than treated as
   the source of truth itself, that a background sync loop needs to run *once immediately* on
   startup and not just on an interval — these are architectural judgment calls. An agent can
   implement a decision quickly; it can't substitute for having made the right one.
3. **Have real DevOps knowledge to actually ship it.** Docker multi-stage builds, why a Compose
   override needs `!reset` and not `null`, how Terraform's `templatefile()` actually scans a
   file, how Lightsail's billing model differs from EC2's, how to keep secrets out of an image
   while still getting them into a running container — none of this is "describe what you want."
   It's knowing enough about the actual infrastructure to know what to ask for, and to recognize
   when the output is wrong.
4. **Know what it will actually cost to run.** Before this goes in front of a real customer, they
   need a real number, not "it's cheap." For this stack running continuously: ~$12/month for the
   Lightsail instance, ~$0.50/month for image storage, and the only real variable cost —
   Gemini API usage — running roughly **$15-75/month total** depending on how much the team
   actually uses it (light to heavy daily use). Being able to reason about *why* that's the
   number — which parts are fixed infrastructure and which scale with usage — is the same skill
   it's always been.

## What's next
I'm planning to write more about the LangGraph agent design specifically — how the
quantitative/qualitative/synthesis flow is actually structured as a graph, and the MCP protocol
side for anyone wanting to expose their own tools to Claude Desktop or similar clients. Stay
tuned!

----
If you have any feedback on this post, or a topic you'd like me to write about, let me know at
[budi.arsana@bungamata.com](mailto:budi.arsana@bungamata.com).
