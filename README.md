You're right to pause here. We have been discussing two separate H1Bot implementations, with different goals, different repositories, and different development stages. The confusion comes from calling both of them H1Bot and discussing them in parallel.

Let me explain the entire picture, from what exists today to what we're trying to build.

# 1. The big picture: two H1Bot projects

H1Bot Core

Phase 16 committed

# Existing OpenCode-derived application

Repository: `D:\h1bot\my-agent`

This is the existing, much larger coding-agent application that we have been rebranding and securing.

Current checkpoint: `3d572d57` on `h1bot/mvp`

H1Bot Startup MVP

Post-B0 remediation

# New local-first durable agent product

Repository: `D:\h1bot\startup-mvp`

This is a separate implementation focused on persistent, recoverable AI agent tasks.

Current checkpoint: B0 implementation and independent engineering review completed; critical blockers identified.

They are not two phases of the same codebase. Phase 16 belongs to Core. B0 belongs to Startup MVP.

# 2. Session 1 — H1Bot Core (Phase 16)

Think of this as the existing OpenCode application that we are transforming into H1Bot.

## What are we building here?

OpenCode is the foundation. We are modifying and securing its existing application rather than writing an entirely new coding agent.

Its broad architecture is:

Frontend

Existing OpenCode-derived user interface

Backend / Core

Sessions · tools · agents · permissions · providers

AI models and integrations

Model providers · MCP · plugins · tools

This is a conceptual view, not a verified package-level architecture diagram.

### What has happened so far?

We went through multiple phases of modifying the existing application.

The most recent work was Phase 16, which focused on security.

Examples of what was implemented:

- Canonical path containment checks.
- Project trust management through `trusted-projects.json`.
- Trust gating for plugins and other project-controlled integrations.
- Child-session permission inheritance.
- Authorization for moving sessions outside a workspace.
- Regression tests for security-sensitive behavior.
- Security documentation and audit records.

### Where exactly are we?

Committed checkpoint

| Item                          | Status         |
| ----------------------------- | -------------- |
| Phase 16 Stage 4/5            | Complete       |
| Commit                        | `3d572d57`     |
| Local commit                  | Yes            |
| Push to remote                | No             |
| Tag / deployment              | No             |
| Internal use                  | GO             |
| Controlled loopback-only beta | Conditional GO |
| Limited beta                  | NO-GO          |
| Public release                | NO-GO          |

There are still security limitations:

- Webfetch SSRF exposure.
- Bearer token in query string.
- No main API Host validation.
- No authentication rate limiting/lockout.
- Other documented trust and test-coverage limitations.

So Phase 16 being committed does not mean the application is completely secure.

### Is the frontend being built here?

No new frontend is currently being built from scratch.

The existing OpenCode-derived frontend is already part of the application. The work we've discussed is predominantly backend/core security, behavior, testing, and documentation.

The controlled beta is intended to test this existing application, not a newly designed H1Bot frontend.

The next proposed work in this session is beta documentation, not a new UI.

# 3. Session 2 — H1Bot Startup MVP (B0)

This is the more interesting new product direction.

Here, we are not primarily trying to rebrand OpenCode.

We are building a new application around one central problem:

AI agents should be able to execute long-running tasks, survive interruptions, recover safely, and preserve their progress.

For example:

You ask H1Bot:

> Build a small web application with authentication, database integration, and tests.

Suppose your laptop shuts down halfway through.

A normal chat-based agent might lose its execution context, repeat work, or leave you unsure about what actually happened.

The Startup MVP is intended to make task execution durable.

## What is the architecture?

This is the architecture we selected for the Startup MVP:

React + Vite

Frontend · task creation · progress · approvals · history

HTTP API / polling

Fastify + TypeScript

Local backend API · validation · service coordination

Durable execution runner

Task lifecycle · checkpoints · recovery · retries · approvals

SQLite

Persistent task state and history

@opencode/ai

Model and provider communication

The runner uses SQLite for durable state and the AI adapter for model communication.

### What does each component do?

| Component      | Responsibility                                       |
| -------------- | ---------------------------------------------------- |
| React + Vite   | What the user sees and interacts with                |
| Fastify        | Receives frontend requests and exposes the local API |
| Durable runner | Actually manages task execution and recovery         |
| SQLite         | Remembers tasks, steps, events and execution state   |
| `@opencode/ai` | Talks to the AI model and handles model interactions |
| Bun            | JavaScript/TypeScript runtime and packaging          |

One important distinction: `@opencode/ai` is being reused as a model communication library. We are not importing the entire OpenCode application core into this new MVP.

## What has B0 achieved?

B0 was the initial implementation milestone.

The key engineering achievement was testing crash recovery instead of simply testing whether an agent can answer a prompt.

Four scenarios were tested:

| Scenario | What happens                              | Result                                       |
| -------- | ----------------------------------------- | -------------------------------------------- |
| A        | Crash before sending a model request      | Recovery asks once                           |
| B        | Request sent, response never arrives      | Outcome treated as uncertain                 |
| C        | Tool acts, but result is not recorded     | Recovery avoids blindly repeating the action |
| D        | Step recorded, crash before next question | Recovery preserves completed step            |

The reported product test suite passed:

80 passed · 0 failed

These tests used a real SQLite file, a separate OS process, a local provider stub and actual HTTP requests.

That is meaningful progress. But it does not prove universal exactly-once execution.

# 4. Why are we still working on B0 if the tests passed?

Because after B0, we commissioned an independent engineering review.

That review found issues that ordinary product tests did not catch.

Four major external-beta blockers were discovered:

1\. Local API security

Critical

A malicious website could interact with the local API and cause the agent to execute commands.

The review reproduced a request that created a job executing a shell command outside the selected folder.

Planned fix: Host validation, Origin/Sec-Fetch-Site checks, per-launch authorization tokens, and browser security headers.

2\. Concurrent execution

Two H1Bot processes using the same database could interfere with each other. One could park a step while the other was executing it.

Planned fix: Single-instance enforcement, job leases and compare-and-set coordination.

3\. Incomplete model output

The model could return truncated output, but the application could still display `Finished.`

Planned fix: Explicit incomplete-output states and attention codes.

4\. Concurrent database migrations

Two processes starting against a fresh database could race during migration.

Planned fix: Serialize migration execution using a database lock.

There are also lower-priority issues, including missing retry behavior and unsupported multiple tool calls.

## Where is Startup MVP now?

The implementation plan is in:

`D:\h1bot\startup-mvp\planning\POST_B0_IMPLEMENTATION_PLAN.md`

You approved the plan and authorized implementation, starting with security and failing-first tests.

The planned work is:

| Step | Work                                       | Status in the latest context                       |
| ---- | ------------------------------------------ | -------------------------------------------------- |
| 1    | Local API security                         | Authorized; implementation result not yet reported |
| 2    | Incomplete model output                    | Planned                                            |
| 3    | Parallel tool calls                        | Planned                                            |
| 4    | Bounded model retries                      | Planned                                            |
| 5    | Error-message cleanup                      | Planned                                            |
| 6    | Live provider test                         | Not run; environment variables missing             |
| 7    | Leases, migration locking, single instance | Planned                                            |
| 8–9  | Executable packaging and acceptance test   | Planned                                            |

Important: The latest context contains your authorization, not proof that these changes have actually been implemented. I don't want to tell you that the backend is fixed when we haven't received the implementation report.

# 5. Are we building a frontend and backend?

Yes, but there are two different situations.

| Area                | H1Bot Core                           | H1Bot Startup MVP                                                     |
| ------------------- | ------------------------------------ | --------------------------------------------------------------------- |
| Frontend            | Existing OpenCode-derived UI         | React + Vite UI                                                       |
| Backend             | Existing OpenCode-derived backend    | New Fastify backend                                                   |
| Agent execution     | Existing OpenCode execution system   | New durable task runner                                               |
| Database            | Existing core's storage architecture | SQLite                                                                |
| Main focus now      | Security and controlled beta         | Reliability, security and packaging                                   |
| New UI being built? | Not currently established            | Frontend exists in the B0 architecture; further work must be verified |
| Release state       | Conditional loopback beta            | Not ready for external beta                                           |

The key thing to understand:

- Core: We are modifying an existing, substantial coding-agent application.
- Startup MVP: We are building a smaller, independent product with its own frontend, backend and execution engine.

Neither should be mistaken for the other.

# 6. What are we ultimately heading towards?

My understanding of the product direction is this:

# H1Bot

## A local-first AI agent that can keep working, recover and continue.

[Niebex - Agentic AI & Software Engineering](https://images.openai.com/static-rsc-4/U-F2EL7ZUUN9SJzzPhdUHZ-hixI7oV_ctXgVVQHCArSrnjo5ULTa7ePMUd0IAxUht-dkCMBRiVVlDB-sId_Ocvb-o3u00RK_cc2f4JTdBX4aosVLkkOlBDdMBTfpQNJszDNlaDxFHf7ZKY9PPADJMRyIksoXRWHWx4Pi2XBSHlI?purpose=inline)

[niebex.com](https://niebex.com/?utm_source=chatgpt.com)

The intended user experience

1. Open H1Bot on your computer.
2. Connect your preferred AI model using your own API key.
3. Give it a substantial task.
4. The agent executes the task and persists its progress.
5. You can inspect its steps, events and pending approvals.
6. If execution is interrupted, H1Bot recovers from recorded state rather than blindly starting over.
7. The user remains in control of risky operations.

The long-term idea is not just another ChatGPT wrapper or another coding-agent UI.

It is a durable execution layer for AI agents.

The product should be able to handle tasks that take a long time, survive process restarts, maintain a trustworthy execution history and avoid repeating dangerous operations.

There is still a product decision to validate: whether the Startup MVP eventually becomes the primary H1Bot product, or whether we combine its durable execution ideas with the existing OpenCode-derived Core. That combination has not been established as completed architecture.

# 7. What should happen next?

Here's how I would organize the work so we stop getting confused.

Track A — Existing Core

Phase 16 checkpoint complete

- Prepare controlled beta documentation.
- Keep loopback-only restrictions.
- Run controlled testing with technically capable testers.
- Track the remaining security limitations.
- Do not push or deploy until explicitly approved.

Track B — Startup MVP

Engineering remediation

- Complete the local API security fix first.
- Show failing-first and passing regression tests.
- Fix incomplete model output and parallel tool calls.
- Implement safe retries.
- Fix concurrency, leases and migrations.
- Fix executable packaging.
- Run the live-provider test only after credentials are configured.
- Reassess beta readiness after all blockers are addressed.

## 8. The most important thing: which one is the actual product?

My recommendation is to keep both projects separate for now.

The Core is a mature existing codebase with a lot of functionality and a security-remediation history.

The Startup MVP is a focused experiment to prove the durable-agent execution model.

Do not merge them prematurely. First establish that Startup MVP can reliably execute, recover, secure and package a task-running agent. Then decide whether its durable execution layer should integrate with Core or become the foundation of a new product.

## 9. How to keep track of everything going forward

I suggest maintaining one project-level status document outside both repositories:

`D:\h1bot\H1BOT_MASTER_STATUS.md`

It should contain:

- Project overview and product thesis.
- Core vs Startup MVP distinction.
- Repository paths and branches.
- Current phase and latest commit for each.
- Frontend/backend architecture for each.
- Completed work, pending work and blockers.
- Test evidence and release readiness.
- Decisions made and decisions awaiting approval.
- Next milestone for each track.

That way, you won't have to reconstruct the whole project from two separate ChatGPT sessions.

My final assessment: Core has reached a controlled-beta preparation checkpoint. Startup MVP has proven an initial durable-execution concept, but its independent review found critical issues that must be fixed before external beta. The next major engineering effort should be Startup MVP's security remediation, while Core remains on its separate controlled-beta track.

One thing I would not do yet: announce H1Bot as a finished product. We're still validating the architecture and the reliability/security foundations.
