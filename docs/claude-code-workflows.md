# Claude Code Dynamic Workflows — Reference Notes

Summary of the official docs at <https://code.claude.com/docs/en/workflows>.
Check the source page for the latest details; version-specific behavior is noted inline.

## What a workflow is

A **dynamic workflow** is a JavaScript script, written by Claude, that orchestrates many
[subagents](https://code.claude.com/docs/en/sub-agents) at once. A runtime runs it in the background,
so your session stays responsive.

Use one when:

- the task needs more agents than one conversation can coordinate (codebase-wide bug sweeps,
  500-file migrations), or
- you want the orchestration written down as a script you can read, save and rerun, or
- you want a quality pattern built in, such as agents adversarially checking each other's findings
  or drafting a plan from several angles and comparing them.

Availability: all paid plans, Anthropic API, Amazon Bedrock, Google Cloud Agent Platform and
Microsoft Foundry. On Pro, turn on the **Dynamic workflows** row in `/config`.

### How it compares

|                        | Subagents            | Skills               | Agent teams              | Workflows                 |
| :--------------------- | :------------------- | :------------------- | :----------------------- | :------------------------ |
| What it is             | A worker Claude spawns | Instructions Claude follows | Lead agent supervising peers | Script the runtime runs |
| Who picks the next step | Claude, each turn   | Claude, per prompt   | Lead agent, each turn    | The script                |
| Intermediate results   | Claude's context     | Claude's context     | Shared task list         | Script variables          |
| What's repeatable      | Worker definition    | Instructions         | Team definition          | The orchestration itself  |
| Scale                  | A few per turn       | Same as subagents    | A handful of peers       | Dozens to hundreds per run |
| Interruption           | Restarts the turn    | Restarts the turn    | Teammates keep running   | Resumable in the same session |

The main difference: in a workflow, the plan is code. Claude's context only holds the final answer.

## Getting started: `/deep-research`

`/deep-research` is the bundled workflow, and it only runs when you invoke it. It needs the WebSearch tool.

1. Run it: `/deep-research What changed in the Node.js permission model between v20 and v22?`
2. Approve the launch when asked (see [Approval](#approval)).
3. Watch it: run `/workflows`, or focus the progress line in the task panel (down arrow, then Enter).
4. Read the cited report. Claims that failed cross-checking are removed. Claims the verifiers
   couldn't check (rate limits, API errors) are marked **unverified**, not refuted.

## Starting a workflow

| Method | How |
| :-- | :-- |
| Keyword | Put `ultracode` in your prompt, e.g. `ultracode: audit every API endpoint under src/routes/ for missing auth checks` |
| Plain request | Say "use a workflow" / "run a workflow" |
| Session-wide | `/effort ultracode` (or launch with `claude --effort ultracode`, which also sets effort to `xhigh`). Claude then decides when a task needs a workflow |
| Saved/bundled command | `/<name>`, e.g. `/deep-research` |

The keyword only changes how Claude structures the work. Agent tool calls still go through the
normal permission checks and sandboxing.

**Dismissing the keyword:** press `Option+W` (macOS) or `Alt+W` (Windows/Linux), or press backspace
right after it. To disable it entirely, turn off **Ultracode keyword trigger** in `/config`.

**Where the keyword counts:** only in prompts you type yourself: the interactive prompt, IDE panel,
Remote Control client, or an Agent SDK app that marks input as `origin: { kind: "human" }`.
It does **not** trigger from `-p` prompts, unmarked SDK prompts, scheduled tasks, webhook payloads
or relayed PR comments. Before v2.1.210 it triggered from all of these.

### Ultracode mode

- Applies at the session's current effort level. One request can become several workflows in a row
  (understand → change → verify), so it uses more tokens and time, and on subscription plans it
  reaches usage limits sooner.
- While ultracode is on: no `Large workflow` warning, the concurrent subagent limit isn't enforced
  for Agent-tool subagents, and auto mode skips the first-launch approval.
- Lasts for the session. To make it the default, set the `ultracode` setting. Turn it off with
  `/effort ultracode off`. In the `/effort` slider, `Tab` flips the toggle and `Enter` applies it.

## Approval

CLI prompt options: **Yes, run it** · **Yes, and don't ask again for `<name>` in `<path>`**
(offered only for named bundled, saved or plugin workflows) · **View raw script** · **No**.
`Ctrl+G` opens the script in your editor. Press `Tab` on Yes/No to add a comment.

| Permission mode | When you're prompted |
| :-- | :-- |
| Auto | First launch only; any Yes records consent. Skipped entirely with ultracode on |
| Manual / accept edits | Every run, unless "don't ask again" was chosen for that workflow in this project |
| Bypass permissions | Never; the run starts immediately |
| `claude -p`, Agent SDK | Never prompted. Evaluated like any tool call |

To allow launches in `-p` or the SDK, use one of:

- an allow rule: `Workflow` (all) or `Workflow(<name>)` (one saved workflow)
- auto mode (the classifier can approve it)
- bypass permissions mode
- a `PreToolUse` hook returning `"allow"` (not enough on its own in plan mode)
- `--permission-prompt-tool` or an SDK `canUseTool` callback

Desktop app: an approval card with **Once / Always / Deny**. Progress appears in the Background tasks pane.

Spawned agents use your permission rules. Add the tools they need to your allow rules before a long run so it isn't held up by prompts.

## Monitoring: `/workflows`

`/workflows` lists running and completed runs. If there's only one run, it opens that run directly.
Select a run and press `x` to stop it from the list.

| Key | Action |
| :-- | :-- |
| `↑` / `↓` | Select phase or agent |
| `Enter` / `→` | Drill in; in detail view, expand/collapse |
| `Esc` / `←` | Back out |
| `j` / `k` | Scroll agent detail |
| `f` | Cycle status filter |
| `p` | Pause / resume run |
| `x` | Stop selected agent (or whole run when focused on the run) |
| `r` | Restart selected running agent |
| `s` | Save the run's script as a command |

Agent detail shows the agent's prompt, recent tool calls with their state, its result, and its own task list if it keeps one.

## Saving and sharing

Save a run with `/workflows` → select the run → `s`. `Tab` switches the location:

- **Project:** `.claude/workflows/`, shared with the repo
- **Personal:** `~/.claude/workflows/`, or `$CLAUDE_CONFIG_DIR/workflows/` if that's set

Saved workflows run as `/<name>` and appear in `/` autocomplete.

- **Symlinks** (v2.1.216+): for the project location, saving is refused if `.claude`, `.claude/workflows` or the
  target file is a symlink. For the personal location, it's refused only if the target file is a symlink.
- **Monorepos:** saving writes to the nearest existing `.claude/workflows/` between your working directory and
  the repo root (or the repo root if none exists). Workflows load from every such directory, and the nearest one wins on name clashes.
- **Name clash:** a project workflow beats a personal one with the same name.
- **Plugins:** put scripts in a `workflows/` directory at the plugin root, or set the `workflows` manifest field.
  They're namespaced, e.g. `/acme-tools:release-audit`.
- **Input:** saved workflows read a global `args`, passed as structured data (arrays and objects, no parsing needed). It's
  `undefined` if omitted. Example: `Run /triage-issues on issues 1024, 1025, and 1030`.

## Script anatomy

```javascript
export const meta = {
  name: 'audit-routes',
  description: 'Audit every route handler for missing auth checks',
}

const found = await agent('List every .ts file under src/routes/.', {
  schema: { type: 'object', required: ['files'], properties: { files: { type: 'array', items: { type: 'string' } } } },
})

const audits = await pipeline(found.files, file =>
  agent(`Audit ${file} for missing authentication checks.`, { label: file }),
)

return audits.filter(Boolean)
```

| Primitive | Purpose |
| :-- | :-- |
| `agent(prompt, opts)` | Spawn one subagent. Resolves to `null` if it's stopped or hits an unrecoverable error |
| `pipeline(items, fn)` | One agent per item. Keeps `null` entries, so filter them out |
| `parallel(tasks)` | Run agent tasks at the same time and wait for all of them |
| `phase(title)` | Group the agents that follow under a title in the progress view |
| `log(msg)` | Show a message above the phases |
| `args` | Input passed to a saved workflow |

Useful `agent()` options: `schema` (structured JSON output), `label`, `stallMs`.

- **Schema:** checked before the agent starts. A schema that provably contradicts itself (e.g. a `required` key
  excluded by `additionalProperties: false`) fails immediately. Output is validated up to 5 attempts
  (`MAX_STRUCTURED_OUTPUT_RETRIES`).
- **Auto mode:** prompts the script passes to `agent()` aren't treated as requests from you.

### Editing rules

- Run the `/workflow-authoring` bundled skill (v2.1.248+) first to load the reference Claude works from.
- After editing, run `/reload-skills`, then `/<name>`.
- `export const meta` must be the **first** statement and a plain object literal with `name` and `description`.
  Variables, function calls or spreads in it remove the command from autocomplete.
- Each `meta.phases` entry must match a `phase()` title exactly. A `phase()` title with no entry gets its own group.
- `Date.now()`, `Math.random()` and `new Date()` with no arguments **throw**, so that a resumed run repeats the same calls. Pass
  timestamps in through `args`.
- `import()` is not allowed. Put work that needs a library inside an agent's task.

## Runtime

- Runs in an isolated environment. The script has no direct filesystem or shell access; agents do the reading, writing and running.
- Each run's script is written under the session's directory in `~/.claude/projects/`. You can read it, diff it, edit it and ask Claude to relaunch it.
- Claude can only launch scripts the session is allowed to read. Use `/add-dir` or a Read allow rule for scripts elsewhere.

### Limits

| Limit | Value |
| :-- | :-- |
| Mid-run user input | None. Runs pause only for permission prompts and usage-limit waits. For sign-off between stages, use separate workflows |
| Concurrent agents | 16 by default (fewer on CPU-limited hosts). Set `CLAUDE_CODE_WORKFLOW_MAX_CONCURRENT_AGENTS` from 1–256 (v2.1.269+) |
| Items per `parallel()` / `pipeline()` | 4,096. Longer lists are rejected with an error |
| Agents per run | 1,000 |

### Prompt caching

Agents with the same model, effort, agent type, tools, schema and working directory share a cache prefix.
In a fan-out, all but the first matching agent wait until the first one's response starts
(up to `CLAUDE_CODE_WORKFLOW_PREFIX_STAGGER_MS`, default 5000; `0` disables the wait). The cache lasts 5 minutes by default.
Set `subagentPromptCacheTtl: "1h"` for an hour; 1-hour cache writes are billed at a higher rate.

## Managing runs

### Resuming

- **Paused:** `/workflows` → select → `p`.
- **Stopped:** ask Claude to relaunch it with the same script. This is refused until the old agents have exited.
- **Replay rules:** agents replay in the order they started.
  - Completed agents return their cached result, until the first agent whose prompt changed. That agent and every one after it run again.
  - Agents still running at stop time start over. Stopping the run doesn't count as a failure.
  - A failed agent runs again, and so does every agent that started after it. Stopping a single agent with `x` counts as a failure.
  - Example: A, B, C, D where B failed → A comes from cache; B, C and D run again.
- **Leaving the session:** if you background the session, or choose "Move to background and exit" with agent view on, the run continues.
  If you choose "Exit and stop tasks", results are kept and `claude --resume` can replay them. A fresh session starts over.
- **Cloud sessions:** results are saved with the conversation history and survive VM reclaim.
- If the saved results are missing, relaunching fails with `nothing to resume`. Ask Claude to start a new run.

### Usage limits (v2.1.271+)

When an agent hits your claude.ai usage limit, the run pauses: affected agents wait, no new agents start, and the run resumes after the reset.
This only happens if all of these are true: the session is interactive with a claude.ai subscription (not `-p`, the SDK, background sessions,
Remote Control or teammate sessions), `autoContinueAtUsageLimit` is on, the limit resets within 24 hours,
and the run hasn't already waited twice. Otherwise the agent fails.

### Stalled agents

An agent that stops producing output restarts from the same prompt, shown as `(retry 1)` / `attempt 2 (stalled)`. It gets up to 5 restarts,
including manual `r` restarts. Files the stalled attempt changed stay changed, and its tokens still count.
If every attempt fails, you'll see one of these errors:

- `agent stalled on all 6 attempts`: lengthen the stall window
- `agent lost its reply on all 6 attempts`: the streaming idle watchdog ended the response; see `CLAUDE_STREAM_IDLE_TIMEOUT_MS`
- `agent abandoned after 6 attempts`: the attempts failed in different ways, listed in the error

To lengthen the window, pass `stallMs` on one `agent()` call, or set `CLAUDE_ASYNC_AGENT_STALL_TIMEOUT_MS` for every agent.
A failure inside `parallel()` / `pipeline()` leaves `null` and the run continues. A failure in a directly awaited call ends the run.

## Cost control

- Runs count toward plan usage and rate limits, and can use far more tokens than doing the same task in conversation.
- Try a small slice first (one directory, a narrow question). `/workflows` shows token usage per agent, and you can stop at any time.
- A **`Large workflow`** warning appears when a run schedules more than 25 agents or is projected to pass 1.5M tokens. It's advisory only.
  Choosing a size guideline yourself replaces the 25 threshold. Ultracode hides the warning.
- Agent models are chosen the same way as for subagents. A model named in the script counts as the per-invocation model. If nothing sets one, the session model is used.
  Check `/model` before big runs, and ask Claude to use smaller models for easy stages.
  An `availableModels` allowlist substitutes blocked models.

### Size guideline (v2.1.219+)

| Value | Target agent count |
| :-- | :-- |
| `unrestricted` | Sized to the task |
| `small` | Fewer than 5 |
| `medium` | Fewer than 10 (default) |
| `large` | Fewer than 50 |

The default is `small` on Pro with v2.1.271+. Set it with `/config workflowSizeGuideline=small` or the `workflowSizeGuideline` settings key.
A settings file takes precedence over `/config` and hides the `/config` row. The guideline is advice to Claude, not a cap, and applies from the next prompt.

## Turning workflows off

- For yourself: the **Dynamic workflows** toggle in `/config`, `"disableWorkflows": true` in `~/.claude/settings.json`, or
  `CLAUDE_CODE_DISABLE_WORKFLOWS=1`.
- For an organization: `"disableWorkflows": true` in managed settings, or the Claude Code admin settings page.
- Effect: `/workflows`, workflow commands, `/workflow-authoring`, the `ultracode` keyword and the Ultracode toggle all
  go away. Runs already in progress continue. No setting disables ultracode on its own; an effort cap only lowers its effort level.

## Example prompts

| Pattern | Prompt |
| :-- | :-- |
| Audit many files | `use a workflow to audit every route handler under src/routes/ for missing authentication checks, and adversarially verify each finding before reporting it` |
| Fix until passing | `use a workflow to run npx tsc --noEmit and keep fixing the reported errors until the type check passes or two rounds in a row make no progress` |
| Parallel migration | `use a workflow to migrate every component under src/components/ from JavaScript to TypeScript, working on each file in its own isolated copy` |
| Review + summarize | `use a workflow to review every file changed in this PR for correctness issues, then merge the per-file findings into one ranked summary` |
| Multi-source research | `use a workflow to research how our three competitors handle rate limiting: read their public docs and recent changelog entries in parallel, then compare the approaches` |
| Search until saturated | `use a workflow to find flaky tests in this repo: run the suite repeatedly, record which tests fail intermittently, and stop once two rounds in a row find nothing new` |

## Related

- [Run agents in parallel](https://code.claude.com/docs/en/agents)
- [Create custom subagents](https://code.claude.com/docs/en/sub-agents)
- [Manage costs](https://code.claude.com/docs/en/costs)
