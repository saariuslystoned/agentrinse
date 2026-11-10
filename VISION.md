# Vision: AgentRinse for agent fleets

Status: draft proposal for upstream discussion. Nothing here changes
AgentRinse's safety model, roadmap or non-goals; it proposes a user and three
capabilities for the maintainer to accept, change or decline.

## The user

AgentRinse's product spec already names a high-concurrency agent developer whose
coding agents leave worktrees, caches, logs and runtimes behind. This proposal
extends that user to its heaviest form:

> **One person running their own agent fleet.** Tens of concurrent agent
> sessions (Codex, Claude, Cursor and others) spread across half a dozen
> workstation-class machines: several large-memory laptops and a couple of GPU
> boxes, one of which stays on as the gateway while the rest act as nodes. The
> repositories are monorepo-scale, and disk and CPU fill faster than any human
> can review cleanup.

The usual first answer is an always-on cleanup agent per machine. In practice
that agent falls behind: it cannot remove worktrees and dependency installs as
fast as the fleet creates them.

What that looks like in practice, from one such workstation over a single day:

- about 1,300 linked worktrees, with roughly 80 new ones a day, most holding a
  full dependency install of the same repository;
- around 200 GiB of new worktree allocations in 24 hours, and free space falling
  10–20 GiB an hour while hand-approved cleanup batches ran;
- load averages above 300 on 18 cores, with routine `git status` calls taking
  tens of seconds to minutes;
- hand-curated cleanup that freed hundreds of GiB over several days and still
  lost ground.

For this user, cleanup has to happen **when work is done**, not after an age
window, and it has to stay trustworthy at that scale. AgentRinse's refusal model
(prove scope, identity, inactivity and effect, or protect) is exactly the right
foundation. The gaps below come from where that model currently stops working
for this user, not from wanting it loosened.

## Pillar 1: correct at fleet scale

AgentRinse should give the same answer on a 65,000-file monorepo on a saturated
machine as on a small repository on an idle laptop.

- Git listings are streamed rather than captured into fixed output buffers, so
  repository size never decides whether a worktree is inventoried.
- Every time, size and entry budget either completes or produces an explicit,
  named hold for the affected resource. Running out of budget never silently
  drops a repository or a worktree from the audit.
- Mutation-time probes honor the configured budgets (for example process
  ownership), so a busy machine yields a clear "not proven, held" rather than a
  skipped apply.

Today a large repository can fail inspection as a whole (an index listing larger
than the fixed buffer), and a loaded machine can turn every artifact into
`ARTIFACT_PROCESS_UNKNOWN`. Both are correct refusals. The goal is that they
become *precise* refusals, and that the common case succeeds.

## Pillar 2: "done" as an input, never as authority

Fleet operators already know when a task finishes: a PR merged, a session ended,
an orchestrator recorded completion. AgentRinse should accept an explicit,
attributable completion signal from such an orchestrator:

- the signal names the workspace, the owner that completed it and the
  assignment generation it applies to;
- a signal from a superseded owner or generation is ignored;
- the signal may shorten the age threshold for that workspace;
- it never replaces AgentRinse's own proof: cleanliness, reachability, process
  ownership and provider protection are still re-checked before every mutation,
  under the existing lock and revalidation rules.

This keeps the decision of *what is safe* inside AgentRinse while letting the
operator's workflow say *what is finished*. It pairs naturally with the practice
of creating workspaces as copy-on-write snapshots (APFS clones or images, btrfs
snapshots) and removing each one when its work is done. This proposal does
not ask AgentRinse to create workspaces; for the workspaces it can identify, it
could make the "remove when done" half provable.

## Pillar 3: protection based on live sessions

The product spec already intends provider protection to follow sessions that are
"live, recent, pinned, or provider-protected". Today the implementation protects
every worktree that provider metadata references. In a fleet, nearly every
worktree has been touched by some agent session, so reference-based protection
protects everything and nothing is ever cleanable.

This pillar brings the implementation in line with the spec's own intent, keying
provider protection on **liveness**:

- protect a workspace while a provider session for it is running, or was active
  within a configured recency window;
- treat unreadable or ambiguous provider state as protection (unchanged);
- report the specific protecting session so the operator can see why.

## Fit with the existing safety model

None of the pillars relaxes an existing invariant. AgentRinse still never kills
a process, never deletes branches or stashes, never follows symlinks,
never removes the current or main worktree, never touches transcripts or
credentials, and still quarantines worktrees before purge. Each pillar either
makes an existing proof possible at scale or adds a signal that AgentRinse
verifies itself.

## How this will be pursued

In small, evidence-backed pull requests, each one grounded in traces from a real
fleet rather than in speculation:

- every proposed change starts from an observed failure or refusal on real
  machines, reproduced on a synthetic repository before it is proposed;
- audit evidence is shared with `agentrinse audit --json --redact`, which
  replaces paths, removes host fields and salts identifiers per report;
- each pull request stays inside AgentRinse's existing safety model and arrives
  with the tests and proof the repository already requires.

## Open questions

- **Across lots of hardware.** Today each machine is audited and cleaned on its
  own. A fleet operator also wants one policy applied consistently across
  machines, a combined view of what each machine is holding and why, scheduled
  sweeps, load-aware job admission and shared dependency baselines.
  AgentRinse's 1.0 non-goals exclude a background daemon and silent recurring
  deletion. Whether any of this belongs in AgentRinse, in a
  separate companion that only calls its audit/plan/apply, or nowhere, is
  intentionally left open until the maintainer's direction is clear.
- **The primary user.** This draft proposes the fleet operator as a named user.
  If the maintainer prefers to keep a single-developer focus, the three pillars
  still stand on their own as correctness improvements.
