# Data Access Request Triage Agent (Sample / Demo)

**Day 5** of the "Agentic AI in Data Engineering" series. Day 1
([pipeline-incident-agent](https://github.com/NANDAN-CREATOR/pipeline-incident-agent))
was **reactive**. Day 2
([data-quality-monitoring-agent](https://github.com/NANDAN-CREATOR/data-quality-monitoring-agent))
was **proactive**. Day 3
([schema-migration-coordinator-agent](https://github.com/NANDAN-CREATOR/schema-migration-coordinator-agent))
was **coordination and planning**. Day 4
([pipeline-cost-optimizer-agent](https://github.com/NANDAN-CREATOR/pipeline-cost-optimizer-agent))
was **optimization under competing constraints**. This agent introduces
a fifth, distinct pattern: **governance under a hard rule**. It triages
a data access request — classifying sensitivity, checking role fit,
applying least privilege — and recommends approval, denial, or
escalation to a human data owner.

It runs entirely on a **local model via [Ollama](https://ollama.com)** —
no cloud API key required.

> **This is a sample, not a production system.** The "production systems"
> it investigates (a data classification catalog, an identity/role
> registry, a role-based access-norms reference, a past-request
> precedent log, a catalog of masked/aggregated alternatives) are
> replaced with small mock backends returning fixed, hand-crafted data
> across four illustrative scenarios. See
> [Adapting This to Real Systems](#adapting-this-to-real-systems) for
> what pointing this at a real environment would actually take.

> **This agent never grants, denies, or modifies any actual permission.**
> There is no tool in its toolkit that touches a real access-control
> system. Every terminal action is a recommendation for a human data
> owner to review and action.

---

## Why This Agent Needs a Different Guardrail Than Days 1–4

Days 1 through 4 were all, in different ways, about **weighing
evidence** — more corroboration made a conclusion more confident, less
evidence meant escalate. This agent introduces something new, and it's
arguably the most important guardrail idea in the whole series so far:
a **categorical rule that no amount of good evidence is allowed to
override.**

Requests touching certain sensitivity tiers (regulated financial data,
health data) must **always** be escalated to a human data owner — not
"usually," not "unless the justification is unusually good." This
repo's third scenario (`high_sensitivity_always_escalate`) is deliberately
built around a genuinely urgent, genuinely plausible-sounding
justification, specifically to test whether the agent lets good-sounding
reasoning talk it out of a hard rule it isn't supposed to reason its way
around at all.

Two more ideas are baked into this agent specifically:

- **Least privilege as a default, not an afterthought.** Even when a
  request should clearly be approved, the agent is required to check
  whether a narrower, masked, or aggregated version of the data would
  satisfy the stated purpose — and recommend that instead of the raw
  access literally requested.
- **Precedent as evidence, not decoration.** Checking how similar past
  requests were handled isn't just for consistency — if an identical
  request pattern was already denied before for a documented reason,
  that's real evidence toward denial now, not something to politely
  ignore because this requester phrased things differently.

---

## What It Demonstrates

Four independent, runnable scenarios, each testing a different terminal
action and a different governance guardrail:

| Scenario | What happens | Correct terminal action |
|---|---|---|
| `clear_cut_low_sensitivity_approval` | Public data, clean role fit, clean precedent | `recommend_approval`, full scope as requested |
| `least_privilege_masked_alternative` | Raw customer PII requested, but an aggregated dataset fully satisfies the stated purpose | `recommend_approval`, but for the **aggregated alternative**, not the raw table |
| `high_sensitivity_always_escalate` | An urgent, convincing justification for regulated payment-card data — the hard-rule test | `escalate_to_data_owner`, no exceptions |
| `role_mismatch_precedent_denial` | A marketing intern requesting raw salary data; this exact pattern already denied twice before | `recommend_denial`, with an alternative path suggested |

The third scenario is the one worth paying closest attention to: it's
designed so that an agent reasoning purely case-by-case — "urgent VIP
customer, plausible role, some approved precedent for this person's
role elsewhere" — could talk itself into approving something that
policy says must **always** be escalated, full stop.

None of these terminal actions ever grant, deny, or modify anything
automatically — every one just produces a structured recommendation for
a human data owner to review.

---

## Requirements

- Python 3.9+
- [Ollama](https://ollama.com) installed and running locally
- A tool-calling-capable model pulled in Ollama, for example:

  ```bash
  ollama pull llama3.1
  ```

  Other models known to support tool calling in Ollama at the time of
  writing include `qwen2.5`, `mistral-nemo`, and `firefunction-v2` — check
  the [Ollama model library](https://ollama.com/library) for current
  tool-calling support. Watch in particular whether a smaller model
  correctly escalates the `high_sensitivity_always_escalate` scenario
  rather than being persuaded by the urgent justification text to
  approve it directly — that's the single most important signal this
  repo's scenario design is built to surface.

---

## Setup

```bash
git clone https://github.com/NANDAN-CREATOR/data-access-request-triage-agent.git
cd data-access-request-triage-agent
pip install -r requirements.txt

# In a separate terminal:
ollama serve
ollama pull llama3.1
```

## Running the Demo

```bash
python agent.py --scenario clear_cut_low_sensitivity_approval
python agent.py --scenario least_privilege_masked_alternative
python agent.py --scenario high_sensitivity_always_escalate
python agent.py --scenario role_mismatch_precedent_denial
```

Or with a different model:

```bash
python agent.py --scenario high_sensitivity_always_escalate --model qwen2.5
```

You'll see a step-by-step trace: which tool the model calls at each step,
what it returns, and finally one of the three terminal-action records.

**Note on output:** the exact number of steps and their order can vary
between runs and models. What should stay consistent is *which terminal
action* it reaches for each scenario — and, critically, for
`high_sensitivity_always_escalate`, that it escalates rather than
approves, no matter how the model reasons about the urgency in the
stated purpose text.

---

## Project Structure

```
data-access-request-triage-agent/
├── agent.py           # mock tools, four scenarios, schemas, system prompt, agent loop
├── requirements.txt
├── LICENSE
└── README.md
```

Kept as a single file for a sample project like this — see
[Adapting This to Real Systems](#adapting-this-to-real-systems) for how
you'd split it up for a real deployment.

---

## How the Guardrails Work

- **Hard-rule sensitivity tiers, checked first, no exceptions.** The
  system prompt requires calling `get_dataset_classification` before
  anything else, and if the result falls in a defined always-escalate
  set (`restricted-financial`, `restricted-phi`), the agent must
  escalate — this is enforced as an explicit rule in the prompt, and the
  scenario data is deliberately built to include a compelling-sounding
  justification to test whether it holds under pressure.
- **Role-fit and precedent checks for everything else.** `get_requester_profile`,
  `get_role_access_norms`, and `search_past_similar_requests` are all
  required before a non-hard-rule decision, so a role mismatch or an
  already-denied pattern isn't waved through.
- **Least privilege, checked before any approval.** `get_masked_or_aggregated_alternatives`
  must be checked before recommending approval above public/internal
  sensitivity — a narrower option, if one exists, is recommended instead
  of the raw access as requested.
- **Never grants anything.** There is no tool anywhere in this agent's
  toolkit that can modify a real permission — architecturally, not just
  by instruction.
- **Prompt-injection awareness, applied to requester-supplied text
  specifically.** The system prompt explicitly instructs the model to
  treat the requester's own stated purpose — not just tool outputs — as
  data to evaluate, never as something that can talk it out of a rule.
- **Step limit.** A hard cap (`MAX_STEPS`, default 16) forces an
  escalation rather than an unbounded investigation.

---

## Adapting This to Real Systems

Only the **bodies** of these six read-only functions in `agent.py` need
to change to point this at a real environment:

| Function | Would call, in a real deployment |
|---|---|
| `get_dataset_classification` | Your data catalog's classification/sensitivity tagging system |
| `get_requester_profile` | Your identity provider / HR system (role, team, tenure) |
| `get_role_access_norms` | A documented access-policy reference, or historical access-pattern analytics per role |
| `search_past_similar_requests` | Your access-request/ticketing system's history |
| `get_masked_or_aggregated_alternatives` | Your data catalog, filtered to derived/masked datasets registered against the same source |
| `check_existing_active_grants` | Your access-management system's current grant records |

The tool **schemas**, the **system prompt**, and the **agent loop** don't
need to change. You'd also want to, at minimum:

- Get your legal/compliance/security team to explicitly define and sign
  off on the always-escalate sensitivity tier list — this is a policy
  decision, not something to leave to the agent's own judgment, and it
  should be reviewed and versioned like any other compliance control
- Route recommendations to wherever access requests are actually
  approved (an IAM workflow tool, a ticketing system with a data-owner
  approval step), and never let this agent's output auto-apply anything
- Build a scenario-based test suite exactly like this repo's four
  scenarios, but drawn from your own organization's real past requests,
  specifically including cases with compelling, urgent-sounding
  justifications for sensitive data, to stress-test whether your
  chosen model reliably holds the hard-rule line
- Periodically audit recommendations against actual data-owner decisions
  to catch cases where the agent's role-fit or precedent reasoning
  drifted from what your organization actually considers acceptable

---

## License

MIT — see [LICENSE](LICENSE).
