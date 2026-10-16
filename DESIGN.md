# jev-router: design

Status: design only. Nothing is built yet.

`jev-router` is a small local tool that sits between Claude and TypeSafe AI's Jev model.
It keeps Claude's token use low when Claude asks Jev to make decisions.

## Summary

Jev is a model from TypeSafe AI that answers typed questions about text you give it.
It does not write prose.
You send it a `state` (the evidence) and some questions.
It returns a choice, a score or a yes/no probability.

Jev is cheap.
The reference guide lists input at $0.042 per million tokens, and output is free.
A 6,000-token diff costs about $0.00025 to send.
Claude's tokens around that call cost far more.

Claude pays tokens for four things:

1. It reads the skill and tool text that explains Jev.
2. It writes the question definitions, again on every call.
3. It carries the state, either by reading it or by writing it into the call.
4. It reads the answer, including every probability and the usage fields.

`jev-router` removes most of that.
Claude names a saved question set and points at the state.
The router does the rest outside Claude's context and returns a short answer.

Assumption: you use Claude as an agent (for example Claude Code) and let it call Jev for bounded decisions such as triage, routing, scoring and yes/no checks.
If you use Jev only from your own application code, this tool does not apply.
Your application should call the Jev SDK directly.

## Goals

- Cut Claude's input and output tokens per Jev call by at least 80%. This is a proposed target, to confirm after the baseline run.
- Cut the cost of a many-item job from "Claude reads every item" to "Claude reads a summary".
- Never change what Jev answers. Only the presentation changes, unless you turn on state trimming.
- Keep the Jev key and your files out of Claude's context.
- Measure the saving, so the claim rests on numbers.

## Non-goals

- Making Jev itself cheaper. Trimming state helps a little, but Jev's bill is already small.
- Compressing Claude's own reasoning, or other tools' output.
- Replacing the Jev SDK in production code.
- Answering open-ended questions. Jev is not built for that, and neither is this tool.

## Where Claude's tokens go today

The sizes below are estimates for a typical call.
Phase 0 (see Build plan) replaces them with measured numbers.

| Cost | Who pays | Typical size | Why it is large |
|---|---|---|---|
| Skill and tool text | Claude input, every turn it stays loaded | [measure: size of the `typesafe-ai` skill] | The official skill teaches Claude to write Jev integrations, which is more than a decision helper needs |
| Question definitions | Claude output, every call | 100 to 400 tokens | A choice question with 12 labelled options is long, and Claude rewrites it each time |
| State | Claude input and output | the size of the evidence | Claude reads the file, then writes it into the call |
| Answer | Claude input | 200 to 350 tokens | A choice returns a probability for every option, and the response adds `model` and `usage` |
| Retries and errors | Claude input and output | one turn per failure | Claude sees raw 429 and 529 errors and decides what to do |
| Large jobs | Claude input | items × item size | Claude reads or loops over every item |

Output tokens cost more than input tokens, so cutting what Claude writes saves more than the raw count suggests.

## How the tool works

```text
Claude (Claude Code, or another agent)
   |  3 small tools (MCP) or one CLI command
   |  "ask playbook X about state Y"            ~50 tokens
   v
jev-router  (local process, holds the Jev key)
   |-- State resolver: file, glob, git diff, earlier handle -> text
   |-- State policy: allowed folders, secret scan, size check
   |-- Playbook store: saved questions, checked locally
   |-- Cache: same state + same questions -> same answer, no call
   |-- Jev client: official SDK, retries, rate limits
   |-- Formatter: short answer, expanded only when needed
   |-- Ledger: what Claude saw, and what it would have seen
   v
api.typesafe.ai/v1/systemone
```

### 1. Point at the state; do not carry it

Claude passes a reference, not the text.

| Reference | Meaning |
|---|---|
| `file:src/app.py` | One file |
| `glob:docs/**/*.md` | Several files, joined with headings |
| `git:HEAD~1..HEAD` | A diff, with lock files and generated files dropped |
| `handle:h_3fa9` | State the router resolved earlier in this session |
| `text:...` | Inline text, as a fallback for small input |

The router reads the files, checks the size against Jev's limits and sends the text.
Claude never holds it.
This is the largest single saving.

The `preview` option reports the file list and token count without sending anything.
That way Claude (and you) can see what would leave the machine.

### 2. Save the questions once

A playbook is a file that holds the question definitions.
Claude writes it once, or you write it.
After that, Claude says `playbook="pr-triage"`.

```yaml
# .jev/playbooks/pr-triage.yaml
name: pr-triage
model: jev-1.13.0            # pinned; "jev-latest" works but gets a warning
state:
  max_tokens: 20000
  drop: ["**/*.lock", "**/package-lock.json"]
questions:
  area:
    type: choice
    instructions: Which part of the code does this change mainly touch?
    criteria:
      api: Public HTTP or SDK surface
      storage: Database or file formats
      ui: Screens and styling
      other: None of the above
    accept_at: 0.80          # router-only field: confidence needed to trust the answer
  risk:
    type: score
    instructions: How risky is this change to deploy?
    criteria: [Trivial, Low, Moderate, High, Critical]
    accept_at: 0.70
  needs_migration:
    type: noul
    instructions: Does this change require a data migration?
    accept_at: 0.85
```

- The `type`, `instructions` and `criteria` fields match the Jev request, so the router passes them through unchanged.
- The router checks each playbook before any call: required fields, at most 255 choice options, 2 to 10 score levels. A mistake is caught locally with a precise message, with no failed round trip.
- A playbook file maps directly to the SDK's `Choice`, `Score` and `Noul` objects. Later, it can also generate integration code for your own application.
- Playbooks live in the project (`.jev/playbooks/`), so a team can review and share them. The answer shows the playbook's hash, so a changed playbook is never mistaken for the old one.

### 3. Return a short answer

The default answer is plain text, one line per question.
Plain text costs fewer tokens than JSON.

```text
pr-triage jev-1.13.0 cache:miss state:5.9k
area=api (0.94)   risk=2.4/5 (0.62 REVIEW)   needs_migration=no (0.97)
REVIEW risk: level 2 0.41, level 3 0.38, level 4 0.15
```

- Each answer shows its label and a confidence.
- For a yes/no question, the confidence is `|2p − 1|`, as the reference guide suggests.
- An answer below its `accept_at` threshold is marked `REVIEW`, and only those answers show their top probabilities.
- `detail=full` returns every probability. `format=json` returns the raw Jev response.
- `model` and `usage` go to the ledger, not to Claude, unless Claude asks.
- Only labels from the playbook and numbers reach Claude. Text from the state is not echoed back. This also blocks prompt injection through the state from reaching Claude by way of the answer.

Caution: Jev's confidence measures how concentrated the probabilities are.
It is not a measured chance of being right.
Phase 3 sets each `accept_at` from labelled examples, so the thresholds do not rest on guesses.

### 4. Handle many items outside Claude

When Claude would otherwise read or loop over hundreds of items, it calls `jev_batch`.

```text
jev_batch(playbook="ticket-triage", items="glob:tickets/*.md")
```

The router asks Jev about every item, in parallel and under the rate limits.
It saves every full result to `.jev/runs/<run id>.jsonl`.
Claude receives a summary and the items that need attention:

```text
ticket-triage  300 items  jev-1.13.0  cache:41 hit  run r_8f3a
team:    billing 112 | api 87 | account 64 | other 37
urgent:  yes 23 | no 277
REVIEW (14): t-0042 t-0107 t-0111 ...   use jev_batch(run="r_8f3a", show="REVIEW")
```

- Claude reads about 100 tokens instead of the roughly 120,000 it would need to read 300 tickets of about 400 tokens each.
- The router paces requests at 80% of the stated limits (80 requests per second, 100,000 input tokens per second). That 80% is a proposal. The limits may change, so the router reads `retry-after` on any 429.
- Claude can ask for one group, one item or the `REVIEW` list. It decides whether to read the original text of those few items.

### 5. Cache and share state

- One request carries several questions about the same state, because Jev supports that. The state is sent once.
- The cache key is a hash of the resolved model, the trimmed state and the question definitions. A repeat ask returns `cache:hit` at no Jev cost.
- With a pinned model, entries last until you clear them. With an alias such as `jev-latest`, entries last 1 hour (proposed), because the model behind the alias can change.
- If Claude asks again after editing a file, only the changed state goes out as a new call.

### 6. Keep the tool surface small

Claude needs only three tools.
Each description is about 100 to 150 tokens, and the three together aim for 600 tokens or fewer.

| Tool | Purpose |
|---|---|
| `jev_ask` | One state, one playbook (or inline questions), a short answer |
| `jev_batch` | Many items, a summary, a drill-down into a saved run |
| `jev_playbook` | `list`, `show`, `save` and `help <topic>` for playbooks |

- The long guide stays out of context. Claude reads a topic through `jev_playbook(action="help", topic="choice")` only when it needs one.
- A CLI with the same commands (`jev-router ask ...`) costs no tool text at all. Claude can use it through Bash, and CI can use it too.
- A short block for `CLAUDE.md` tells Claude when to use Jev: bounded decisions about text it already has, never research, arithmetic or open-ended writing.

### 7. Route each request to the cheapest tool that fits

The router can also say whether Jev is the right tool.
It uses fixed rules and no model.

| Request | Route |
|---|---|
| An exact test (a regex, a threshold, a file exists) | Code. Do not call Jev. |
| A bounded decision about supplied text | Jev |
| Research, writing, planning, or a decision that needs facts not in the state | Claude |
| A Jev answer below its threshold on a high-stakes playbook | Claude, with only the `REVIEW` items |

This follows the guidance in the reference guide: keep calculations and side effects in code, use Jev for bounded judgement, and use a reasoning model for the rest.
A playbook can mark itself `stakes: high`. For those, the router never marks an answer as trusted without review, whatever the confidence.

## Failure and safety rules

- **Errors are not answers.** A timeout, 401, 422, 429 or 529 never turns into "no", the first option or a zero score. The router returns one line, such as `ERROR rate_limited retry_after=12s`.
- **Retries stay out of Claude's context.** The router retries 429 and 529 with backoff, up to a limit (proposed: 3 tries or 20 seconds). It does not retry 401 or 422.
- **The key stays in the router.** It reads `TYPESAFE_API_KEY` from the environment. It never prints the key, logs it or returns it.
- **The router reads only allowed folders.** The default is the project root. It always denies `.env*`, `*.pem`, `id_*`, `.git/config` and `.npmrc`.
- **A secret scan runs before sending.** If the state contains something that looks like a key or token, the router stops and says which file and how many matches. It does not print the secret. You can change this policy in `.jev/policy.yaml`.
- **State leaves your machine.** The router sends it to `api.typesafe.ai`, or to the Netlify AI Gateway if you choose that. The `preview` option and the ledger make this visible.
- **State trimming is opt-in.** Dropping files or truncating can change Jev's answers. A playbook turns it on explicitly, and Phase 3 tests it against the untrimmed result.

## Measuring the saving

The ledger is a local SQLite file with one row per call.

| Column | Meaning |
|---|---|
| `claude_wrote` | Tokens in the tool arguments Claude sent |
| `claude_read` | Tokens in the text the router returned |
| `would_have_been` | Estimated tokens if Claude had carried the state, written the questions and read the full response |
| `jev_input_tokens` | From Jev's `usage` field |
| `cache`, `latency_ms`, `playbook_hash`, `model` | For tracing and re-runs |

`jev-router stats` prints the totals and the estimated saving.

Token counts start as an estimate (characters ÷ 3.5).
The benchmark runs calibrate the estimate against the Anthropic token-counting endpoint.
The `would_have_been` column is an estimate, so the benchmark in Phase 3 carries the real claim.

### Benchmark plan

The `benchmark` repo already has what the test needs: an Anthropic client that records usage, a task and grader format, and `stats.wilson` for confidence intervals.
Check first whether its engine can run a multi-turn tool loop. If it cannot, add a small loop.

Build a set of labelled tasks, for example 100 ticket and diff triage cases with known correct labels.
Run three arms on the same tasks:

| Arm | Setup |
|---|---|
| A | Claude with the official `typesafe-ai` skill, writing its own Jev calls |
| B | Claude with `jev-router` |
| C | Claude alone, with no Jev (the reference) |

Record, per task: Claude input and output tokens, number of turns, correctness against the labels, Jev tokens and wall time.
Report the difference between A and B with a Wilson interval on correctness and a median on tokens.
The router passes if B uses fewer Claude tokens than A and its correctness is not lower beyond the interval.

## Build plan

1. **Phase 0: baseline.**
   Measure arm A on a small task set.
   Record the size of the official skill, the size of typical responses, and the turns used.
   Replace the estimates in this document.
2. **Phase 1: core and CLI.**
   State resolver (`file`, `glob`, `git`), playbook loader and validator, Jev client with retries, short formatter, cache, ledger.
   Done when `jev-router ask pr-triage git:HEAD~1..HEAD` prints the short answer.
3. **Phase 2: MCP server and safety.**
   The three tools, folder policy, secret scan, `preview`.
   Done when the three tool descriptions total 600 tokens or fewer, measured.
4. **Phase 3: batch and calibration.**
   `jev_batch`, run files, drill-down, rate limiter.
   Threshold tuning from labelled data.
   Benchmark arms A, B and C.
5. **Later.**
   Chunking for state above the 32,000-token per-question limit (it can lose information, so it stays optional).
   Playbook-to-SDK code generation.
   Use of the Netlify AI Gateway.

## Suggested stack

- **Language:** Python 3.10 or later, run with `uv`, like the `benchmark` repo.
- **Jev access:** the official `typesafe-sdk` package, which has `TypeSafeClient`, `AsyncTypeSafeClient`, `Noul`, `Choice`, `Score` and `RetryPolicy`.
- **MCP server:** the official MCP Python SDK, over stdio.
- **Storage:** SQLite for the cache and ledger. Plain files for playbooks and runs.
- **Versions:** pin every library when you build, and check the current docs then.

## What I could not confirm

I could not open `docs.typesafe.ai` or most news pages from this environment, because outbound access to them is blocked.
The Jev facts in this document come from one third-party reference guide (read in full for the API section) and from search-result summaries.
Check these against the official docs before you build:

- The limits: 64,000 tokens per request, 32,000 per question, 255 choice options, 2 to 10 score levels, 80 requests and 100,000 input tokens per second.
- The price, $0.042 per million input tokens with free output. One listing says $0.04.
- Whether the same request always returns the same answer. The cache design assumes it is stable enough to reuse.
- The size and wording of the official `typesafe-ai` skill. Phase 0 measures it.
- The JavaScript package name, `@typesafe-ai/sdk`. One search result could not confirm it. This design uses the Python SDK.
- Whether Netlify's AI Gateway uses the same request format.

## Decisions for you

1. **MCP, CLI or both?**
   Recommendation: both, on one core. MCP gives Claude native tools. The CLI costs no standing tool text.
2. **Language?**
   Recommendation: Python, to match `benchmark` and use the official SDK.
3. **Answer format?**
   Recommendation: plain text lines by default, JSON on request.
4. **Where do playbooks live?**
   Recommendation: `.jev/playbooks/` in each project, so they are reviewed like code. Allow a personal folder for your own.
5. **Cache by default?**
   Recommendation: yes with a pinned model, 1 hour with an alias.
6. **Where should the benchmark live?**
   Recommendation: as a new pack in `benchmark`, so results use the same charts and statistics.

## Sources

- [TypeSafe Jev reference guide (unofficial gist)](https://gist.github.com/pjburnhill/adf8d28efcad9df037bfdece178ef965): limits, price, request and response fields, SDK names.
- [TypeSafe docs: quick start](https://docs.typesafe.ai/introduction/quickstart) and [Choice primitive](https://docs.typesafe.ai/primitives/choice): seen in search results only.
- [Netlify: TypeSafe Jev in AI Gateway](https://www.netlify.com/changelog/typesafe-jev-ai-gateway/), [Let's Data Science](https://letsdatascience.com/news/typesafe-ai-launches-jev-decision-model-889a38c0), [MarkTechPost](https://www.marktechpost.com/2026/09/19/typesafe-ai-releases-jev/), [Spring AI](https://spring.io/blog/2026/09/21/spring-ai-typesafe-structured-judgment/), [Flavio Copes on using Jev with coding agents](https://flaviocopes.com/jev-coding-agents/): seen in search results only.
