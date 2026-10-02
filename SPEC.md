# Tendless: Specification

Draft. Supersedes the self-hosted design; the Rust orchestrator is retired once the canister passes the same tests.

## 1. Overview

Tendless takes tickets from a team of developers and has agents carry them through the delivery lifecycle: implementing, opening pull requests, and later reviewing, merging, releasing, and deploying.

The orchestrator is a canister on the Internet Computer, so a team shares one Tendless without anyone hosting a server. Everything else runs on developers' machines and connects to the canister; the canister never connects to anything.

- **Orchestrator**: a Motoko canister. Owns tickets, users, providers, workers, runs, logs and configuration, exposes a Candid API, and decides when workers are needed.
- **UI**: a static web app in an asset canister. Calls the orchestrator as the signed-in Internet Identity principal.
- **CLI**: `tl`, a thin wrapper around `icp canister call`. Used by developers and by agents inside workers.
- **Worker Provider**: a process on a developer's machine that starts and stops workers (Docker today) and holds the credentials they need. It belongs to the developer; its workers act as them and work only their tickets.
- **Worker**: a runtime around an agent. Polls the canister for tickets, does the work, reports back.

One canister serves one project. A project may span several repositories.

```
Browser (II) ──▶ UI asset canister ──agent-js──▶ Orchestrator canister ◀──ic-agent── Worker (poll, heartbeat, logs)
tl ──icp canister call─────────────────────────▶        ▲
                                                        │ provider_poll: status up, orders down
                                                Worker Provider ──▶ Worker container (agent + git + tl + icp)
```

## 2. Concepts

- **Project**: the unit of deployment. One orchestrator canister, one UI canister, one or more repositories.
- **Principal**: every caller is an IC principal. Developers are Internet Identity principals, providers and workers have their own key pairs, the admin is the canister's controller.
- **Ticket**: the unit of work. Has a state, a rank, an optional assignee, a description, and a comment thread.
- **Assignee**: who currently holds the ticket. A worker or a human. Acts as the lease.
- **Agent**: the program doing the work (Claude Code, Codex, ...). Every worker runs one agent. Providers advertise the agents they can run; the image behind an agent is the provider's business.
- **Model**: the model an agent runs with. Each provider advertises the models it supports per agent and a default; a ticket may pick one.
- **Owner**: the developer a ticket belongs to. Only their workers pick it up. Nobody works an unowned ticket.
- **Workable state**: a ticket state that has a prompt. Prompts are shared by all agents. Other states are human-only.

## 3. Ticket model

### 3.1 States

| State | Meaning | Assignee |
|---|---|---|
| `todo` | Being refined | None |
| `ready` | Ready to be worked on | None |
| `in_progress` | Being worked on. Workable, so an unassigned one is resumed by the next worker | Worker doing the work, or none if the previous worker died |
| `in_review` | Resulting work is under review | Worker reviewing, if any |
| `failed` | Needs human intervention | None |
| `done` | Acceptance criteria met | None |

Transitions are not restricted by the orchestrator beyond the ACL. Prompts tell workers which state to move a ticket to.

### 3.2 Assignee

- Set to the worker's id when it picks up a ticket.
- Cleared when the worker moves the ticket to another state, except to `in_progress`, which it keeps holding; also cleared when the orchestrator reaps the worker.
- A ticket in a workable state with no assignee is available to be picked up.

### 3.3 Ordering

- Each ticket has a `rank`, a float. Lower rank is served first.
- New tickets get max rank plus one, so they join the back of the queue.
- Reordering is relative: move a ticket before or after another. The orchestrator computes the new rank as the midpoint and renormalizes ranks when a gap gets too small. Clients never set the raw value.
- Poll returns the available, unblocked ticket with the lowest rank.

### 3.4 Access control

- A ticket may be modified by its assignee, the orchestrator, or a human.
- Any worker may create tickets.
- Any worker or human may comment on any ticket.
- Owner: a caller may set it to their own principal, and the current owner may clear it. Nobody can make someone else the owner. A worker acts for its provider's owner (5). Refused while the ticket has an assignee.

### 3.5 Comments

- Each ticket has a comment thread.
- A comment has an author, body, timestamp, and a resolved flag.
- Anyone who can comment may resolve a comment. Resolution marks the comment as no longer relevant.

### 3.6 Fields

- `id`, `title`, `description`, `state`, `rank`, `assignee`, `created_at`, `updated_at`
- `owner`: a developer, by principal. On create: the creating developer, a worker's user, none for the admin. Only the owner's workers pick the ticket up; an unowned one is never handed out. Changing it: 3.4.
- `links`: list of URLs such as pull requests, added by workers
- `comments`
- `relations`: see 3.7
- `blocked`: a `ready` ticket with an unfinished dependency
- `agent`, `model`: optional. `agent` restricts which workers may pick the ticket up; unset means any. `model` overrides the provider's default for that agent. Both must be advertised by one of the owner's providers (5); an unowned ticket accepts neither.
- `runs`: agent runs on the ticket, newest first (4.7)

### 3.7 Relations

Tickets relate to each other; `links` (URLs) is a separate concept.

- `depends_on`: directed. The ticket cannot be worked until the other is finished, meaning `in_review` or `done`.
- `related_to`: symmetric, informational. Stored once, shown on both tickets.

A relation is refused when it duplicates an existing one, points at the ticket itself, or would make `depends_on` cyclic.

A `ready` ticket with an unfinished dependency is blocked: poll never hands it out and the scheduler does not count it as available work. It stays `ready`; nothing changes its state. Only `ready` is blocked; a resumable `in_progress` ticket is handed out regardless.

A ticket lists each relation with the other ticket's id, title and state, as `depends_on`, `blocks` (the other ticket depends on this one) or `related_to`; `depends_on` carries `satisfied`.

## 4. Orchestrator

### 4.1 Responsibilities

- Store tickets, comments, users, providers and their order queues, workers, runs, usage records, worker logs and the project configuration, all in stable storage across upgrades.
- Serve the Candid API the UI, the CLI, providers and workers use.
- Schedule workers by writing orders for providers.
- Reap dead workers.
- Aggregate metrics.

It makes no outbound calls: no HTTPS outcalls, no inter-canister calls. Randomness for ids comes from `raw_rand`, time from the canister clock.

### 4.2 Identity and authorization

Every call is authenticated by the IC; the canister reads `caller`. Anonymous calls are refused.

- **Admin**: a controller of the canister (`Principal.isController`). Approves and revokes users and edits the configuration. There is no admin token.
- **Developer**: an Internet Identity principal derived for the UI canister's origin. Unknown principals are created as `pending` on their first call and may only call `me` until the admin approves them under a name. Revoking a user deletes their providers and stops the providers' workers.
- **Provider**: its own key pair, generated on the provider's machine; the developer registers its principal (5). A provider's calls are accepted only from that principal.
- **Worker**: its own key pair, generated by its provider; the provider reports the worker's principal when it acknowledges the start order (5). Worker calls are accepted only from the bound principal. A worker acts for its provider's owner.

Nothing secret is stored in the canister: no tokens, no sessions. Canister memory is readable by node operators, so this is by design, not just hygiene.

### 4.3 API

Candid methods. Reads are queries, writes are updates. Shapes follow 3.6 and the records below; the `.did` file is the contract and ships with every release (9).

Tickets (developers and workers; ACL per 3.4):
- `list_tickets(filter)` with optional `state`, `assignee`, `owner`, ordered by rank
- `create_ticket(input)`, `get_ticket(id)`
- `update_ticket(id, patch)`: title, description, state, assignee, owner, links, agent, model. `agent` and `model` are rejected unless one of the owner's providers advertised them in its last status.
- `move_ticket(id, #before id | #after id)`
- `list_comments(id)`, `add_comment(id, body)`, `resolve_comment(id, cid)`, `unresolve_comment(id, cid)`
- `add_relation(id, kind, other)`, `remove_relation(id, kind, other)`

Workers (caller must be the bound worker principal):
- `worker_register(version)`: confirms the worker is alive and reports its binary's version, shown on the worker and its runs.
- `worker_heartbeat()`
- `worker_poll(exclude)`: returns the lowest-ranked available, unblocked ticket owned by this worker's user whose `agent` is unset or matches this worker's, and whose `model` is unset or supported by this worker's provider, with the prompt for its current state, the ticket's `model` (may be null), the project's repos, and the id of the run it opens, or nothing. Sets assignee atomically. `exclude` (the ticket the worker just timed out on) makes that ticket last in line.
- `worker_usage(ticket, usage)`: token and cost usage for a ticket, and the model the agent ran with. Ends the worker's open run and records the model on it.
- `worker_logs(run, lines)`: `run` is one of the worker's runs or null. A batch stays under the 2 MB ingress limit.

Logs and workers (developers):
- `worker_log_lines(worker, after)`, `run_log_lines(run, after)`: oldest first, at most 1000 lines per call; `after` supports following live output.
- `list_workers()`: workers with agent, provider and status.
- `list_agents()`: the agents the caller's providers advertised in their last status (a worker's: its user's), each with the union of the models advertised for it.

Providers (5):
- `list_providers()`: every provider with id, owner, name, principal, last status and health: `reachable` (it polled within the timeout), `last_seen`, `last_error`. The last status is kept while a provider is unreachable.
- `add_provider(name, principal)`: adds a provider owned by the calling developer; `name` is unique per owner. Not for the admin or workers.
- `remove_provider(id)`: owner or admin. Stops the provider's workers.
- `provider_poll(status, acks)`: caller must be the provider's principal. Reports the provider's current status and acknowledges orders; returns the orders not yet acknowledged.

Users:
- `me()`: the caller with their `status`.
- `list_users()`: every user for the admin, approved ones for everyone else.
- `approve_user(principal, name)`, `revoke_user(principal)`: admin only.

Configuration (7):
- `get_config()`: the configuration plus the canister's `version`.
- `set_config(config)`: admin only.

Metrics:
- `metrics()`: totals and breakdowns per ticket, per worker, per agent and per model. Tokens in, tokens out, cost, tickets completed, tickets failed.

`version()` is the one method open to anonymous callers.

### 4.4 Lost replies

An update may commit while its reply is lost. Clients do not re-sign such a call: ic-agent polls `request_status` for the original request id until it has the reply, and the IC rejects a duplicate of a signed request within its five-minute window. The API carries no idempotency keys. The worker therefore never issues a second `worker_poll` while one is unresolved.

### 4.5 Scheduler

A recurring timer, every `scheduler_interval` (default 10 s):
1. Take each provider's last status: capacity, workers, advertised agents. A provider that has not polled within `provider_timeout` is unreachable and skipped this pass.
2. Count available tickets per owner, agent and model: owned, unassigned, in a workable state, not blocked (3.7). Tickets with no `agent` form, per owner, a shared pool that any of that owner's idle workers drains.
3. Count idle and busy workers per owner and agent, and which models each can serve from its provider's status.
4. For each worker needed, create a worker record with an id, agent and provider and append a `start` order to that provider's queue. The provider is the one among the owner's with the most free capacity among those advertising the agent and, when the ticket sets one, the model. For an owner's shared pool, when none of their workers is idle, start on their provider with the most free capacity using the first agent it advertises. Start until each owner's agent has one worker per available ticket, capped by `max_workers` and by each provider's remaining capacity.
5. When an owner's agent has no available tickets and their shared pool is empty, append `stop` orders for its idle workers.

A pass must stay within the instruction limit at a few thousand tickets and a few hundred workers.

Later: worker affinity.

### 4.6 Reaper

A recurring timer, every 15 s. A worker that misses heartbeats for longer than `heartbeat_timeout` is marked dead. Any ticket it holds keeps its state and has its assignee cleared, its open run is ended, and a comment on the ticket says the worker disappeared, linking the run's log. Since `in_progress` is workable, the next worker resumes it. A `stop` order is appended for the worker's provider.

There is no retry cap. A ticket that keeps killing workers is caught by humans watching the UI.

### 4.7 Runs and logs

A **run** is one hand-out of a ticket to a worker: opened by poll, ended by the worker's usage report or by the reaper. A ticket lists its runs. Runs carry the worker's agent and, once ended by a usage report, the model it ran with.

A worker ships every line it prints and every line its agent prints to the canister, tagged with the current run or with none (startup, polling, a crash before the first poll). Lines are raw text, truncated at 16 KiB, sent in batches every second or every 100 lines, whichever comes first, so a crash loses at most one batch. Interval, batch size and line limit are worker configuration (`TENDLESS_LOG_INTERVAL`, `TENDLESS_LOG_BATCH`, `TENDLESS_LOG_MAX_LINE`).

Retention: `log_retention` (default 7 days). An hourly timer deletes a run's lines once the run ended that long ago, and run-less lines that old by their own timestamp. Run rows are kept, so a ticket still lists every run; only the text goes.

## 5. Worker Provider

A process on the developer's machine. On first run it creates its identity in icp-cli's identity directory under a configured name, with plaintext storage since it runs unattended, and prints its principal. The developer registers it with `tl provider add <name> <principal>` or in the UI. The provider is reached by nothing: it polls.

A provider belongs to the developer who registered it: its workers act as that developer and work only their tickets. It holds one set of agent credentials, so one provider is one account; the canister never sees them.

A provider is configured with the agents it can run: for each, an image, the models it supports and the default among them. It advertises agents and models in its status; the orchestrator validates tickets and routes work against the union of what providers advertise. To pin work to an account, give that account's provider an agent name no other provider advertises.

Loop, every `poll_interval` (default 5 s): call `provider_poll(status, acks)`.

- `status`: the provider's `version`, total capacity (maximum workers it can run), capacity in use, the workers it believes are running with their agent and status, the advertised agents with their models and default, and anything provider-specific.
- Orders returned: `start { order, worker_id, agent, model }` and `stop { order, worker_id }`. Each carries an order id. Orders are returned until acknowledged, so a lost reply loses nothing and a provider started after orders were issued receives them on its first poll. The provider applies each order once, keyed by order id.
- `acks`: the order ids applied since the last poll. A `start` ack carries the worker's principal; the canister binds it to the worker record and refuses any later rebinding.

Starting a worker: generate a key pair, start the container with these environment variables: canister id and network, worker id, the worker's private key (PEM), agent, the agent's default model, and the credentials the provider is configured with (git token, agent credentials). The provider maps `worker_id` to its own handle (container id, pod name) internally and knows nothing about tickets or repos. Stopping a worker stops the container.

Health in the canister derives from the poll stream: `reachable` while the provider polled within `provider_timeout`, `last_seen` the time of its last poll, `last_error` the last rejected poll.

The scheduler uses capacity from the status as an upper bound alongside `max_workers`.

Implementation: Rust, linking `ic-agent`. It reads its own PEM from icp-cli's identity directory by name; it does not link icp-cli's crates. Docker provider first; later Kubernetes, cloud VMs.

## 6. Worker

### 6.1 Loop

1. Load the key from the environment. Import it for `tl` with `icp identity import --from-pem --storage plaintext`. Register with the orchestrator. Start heartbeating.
2. Poll for a ticket.
3. Receive ticket, prompt, and repo list.
4. Prepare workspace: configure git and gh credentials. Repos are not cloned here; the agent clones what it needs, on demand, and reuses what is already present from earlier tickets.
5. Run the agent through the adapter with the prompt, the ticket's model or the default from its environment, and the agent's `run_timeout`, shipping its output as it arrives (4.7). On timeout, kill the agent, comment on the ticket, and leave it in `in_progress` for another worker to resume.
6. Report usage and the model used. If the agent finished but left the ticket in `in_progress`, move it to `failed` with a comment explaining why. Both comments link to the run's log in the UI.
7. Repeat from 2 until the orchestrator stops the worker.

Heartbeat every 10 s, poll every 5 s.

### 6.2 Agent adapter

```
run(prompt, model, workspace, timeout) -> Outcome { success, summary, links }, Usage { tokens_in, tokens_out, cost, model }
```

The agent updates the ticket itself using the `tl` CLI, which is in the image and pre-configured with the worker's identity. The adapter does not parse agent output to learn the result. It reads the ticket state afterwards.

MVP adapter: Claude Code CLI in non-interactive mode with `--model <model> --output-format stream-json --verbose`, one JSON event per line, usage from the final `result` event; authenticated with a Claude OAuth token rather than an API key. Later: Codex, Pi, others.

### 6.3 Image

Contains the worker binary, the `tl` CLI, `icp`, git, the GitHub CLI, the agent CLI, and the Tendless skill.

### 6.4 Tendless skill

A skill installed in the image, in the agent's skill location, that teaches the agent how to work with the orchestrator: read its ticket including comments, change state, add links, relate tickets, comment, create tickets, all through the `tl` CLI. This is how human guidance left in comments reaches the agent.

### 6.5 Forge

GitHub only. Repos are cloned over HTTPS with the git token and pull requests are opened with the GitHub CLI.

## 7. UI and CLI

### 7.1 UI

A static web app in an asset canister, deployed next to the orchestrator and configured with its canister id. It calls the orchestrator with agent-js as the signed-in Internet Identity principal; there is no session token.

It lists tickets in rank order with blocked ones marked, shows one ticket with comments, relations and runs, allows creating, editing, relating, reordering tickets, setting agent and model, and changing state, shows workers with their agent and provider, and metrics. A ticket's page lets the signed-in developer make themselves its owner unless a worker holds it. A Providers view lists every developer's providers read-only with owner, principal, health, workers and the agents and models each advertises with the default marked, refreshed with the board's poll. The cog opens the configuration (7.3), editable by the admin, and the providers the caller manages: add with a name and principal, remove. A run's log opens at `#/tickets/{id}/runs/{run}` and a worker's at `#/workers/{id}`, both following live by querying with `after`. When the UI knows the agent behind a run or a worker (Claude Code today), its log opens in a pretty view that renders every event as structure, with a toggle to the raw lines.

A pending or revoked user sees a page with their principal and the `tl user approve` command, polling until approved.

The asset canister serves `/.well-known/ii-alternative-origins` listing the login origin icp-cli uses (`cli.id.ai` for `id.ai`), so a developer's CLI identity is the same principal as their browser (7.2).

### 7.2 CLI

`tl` wraps `icp canister call --candid tendless.did` with the same capabilities as the UI. Configured by environment: `TENDLESS_CANISTER`, `ICP_NETWORK`, `TENDLESS_IDENTITY` (an icp-cli identity name). A developer creates their identity once with `icp identity link web --app <ui domain>`, which signs in with Internet Identity in the browser and yields the same principal the UI uses. `tl` renders Candid replies as text or JSON.

`tl ticket logs <id> [--run <n>] [-f]` and `tl worker logs <id> [-f]` print logs, `-f` following until the run ends or the worker dies. `tl provider add <name> <principal>`, `tl provider list` and `tl provider remove <id>` manage the caller's providers. `tl user approve <principal> <name>` and `tl user revoke <principal>` are for the admin. `tl config get` and `tl config set <file>` read and replace the configuration.

### 7.3 Configuration

Lives in the canister, edited in the UI or with `tl config set`. Init arguments carry only the project name. There is no config file.

```
project:   name, repos
scheduler: max_workers, scheduler_interval (10s), provider_timeout (30s)
workers:   heartbeat_timeout (60s), log_retention (7d)
agents:    { name -> run_timeout }
prompts:   { state -> template }
```

The `prompts` table maps states to templates, shared by all agents; `{{ticket.id}}` and `{{ticket.title}}` are substituted. Workers only pick up tickets in states that have a prompt. An agent a provider advertises but `agents` does not list is never scheduled.

The provider's own configuration stays a TOML file on its machine: identity name, poll interval, canister and network, max workers, agents with image, models and default, `worker_env` with the credentials; `provider.secrets.toml` merged over it as before.

## 8. Metrics

Workers report usage per ticket after each agent run. The orchestrator stores raw records and aggregates by ticket, worker, agent and model. Exposed via `metrics()` and the UI.

## 9. Delivery

A GitHub release publishes the orchestrator wasm and `.did`, the UI bundle, the `tl`, worker and provider binaries, and the worker image. A team deploys with `icp deploy` from a checkout or the release bundle into canisters they control, funded by them. One project per deployment. Upgrades are `icp deploy` again; state is in stable storage.

Existing self-hosted instances are not migrated; teams start empty.

Order of work:
1. Candid interface and Motoko canister with the ticket model, users, providers, workers, runs, logs, timers.
2. Worker and provider on `ic-agent`; provider polling with orders; per-worker keys.
3. `tl` over `icp canister call`.
4. UI on agent-js in an asset canister with the alternative-origins file.
5. Demo on `icp network start` with the real binaries; retire the Rust orchestrator.

## 10. Later

- Agent session refresh between tickets, so a long-lived worker starts each ticket with a clean context but warm repos.
- Retry cap: attempt counter that moves a ticket to `failed` after too many interrupted runs.
- Other forges (GitLab, Gitea).
- Worker affinity as a poll parameter.
- Additional workable states: `todo` for refinement, `in_review` for agent review, and states for merge, release, deploy.
- External tracker sync (GitHub Issues, Jira).
- Usage normalization across agents.
- Kubernetes and cloud VM providers.
- Several projects per canister.

## 11. Tech stack

- **Orchestrator**: Motoko, stable storage with enhanced orthogonal persistence, timers. Candid API only; no `http_request`.
- **UI**: plain HTML and JavaScript with agent-js and `@icp-sdk/auth`, in the static-site asset canister deployed by icp-cli.
- **CLI**: Rust, shells out to `icp`.
- **Worker**: Rust with `ic-agent`, in a Docker image with `icp`, the Claude Code CLI, git, the GitHub CLI, `tl`, and the Tendless skill.
- **Docker provider**: Rust with `ic-agent`. Talks to the local Docker daemon.
- **Dev and test**: `icp network start` for a local replica; canister integration tests drive it over `ic-agent` with the real worker and provider; the UI's Playwright suite and `demo/demo.sh` run against it.
