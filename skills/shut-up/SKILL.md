---
name: shut-up
description: Use when writing or editing prose, documentation, code, or your own responses, to cut redundancy, restatement, filler, and defensive cruft. Triggers on symptoms like trailing summaries, comments paraphrasing code, try/catch for impossible errors, single-use helpers, re-stating the user's question, "it's worth noting", duplicated information across prose + bullets + tables.
---

# Shut Up

> **If removing it does not lose information the reader needs, remove it.**

Applies to docs, code, and your own responses. "Information the reader needs" means non-obvious facts, hidden constraints, or the *why* behind a non-obvious choice — not restatement, transitions, throat-clearing, or politeness.

## Delete on Sight — Prose and Docs

| Pattern | Example | Fix |
|---|---|---|
| Preamble | "In this document, we describe..." | Start with content |
| Header restatement | `## Installation` followed by "This section covers installation." | Delete the sentence |
| Throat-clearing | "It is worth noting that...", "Importantly,...", "Note that..." | Delete the phrase; the claim stands alone |
| Trailing summary | "In summary, we have X, Y, Z" after a short doc | Delete |
| Triple-format | Same list written as prose + bullets + table | Pick one |
| Meta-commentary | "As mentioned above...", "We will now discuss..." | Delete |
| Hedge stacks | "It might be possible that perhaps..." | State the claim or do not |
| Parallel bullets saying the same thing | Three bullets restating one idea | Keep one |

## Delete on Sight — Code

| Pattern | Why it is waste | Fix |
|---|---|---|
| Comment restating code | `// increment i` above `i++` | Delete comment |
| Docstring paraphrasing signature | `"""Returns the user by id."""` on `get_user(id) -> User` | Delete, or state a non-obvious invariant |
| Try/catch for impossible errors | Wrapping internal call that cannot throw | Delete handler |
| Validation on trusted input | Re-checking types at an internal boundary | Trust the caller |
| Single-use helper | `def _add_one(x): return x+1` used once | Inline |
| Backwards-compat shim for unreleased code | Re-exports, aliases, `// removed` comments | Delete shim and old name |
| "Added for X" / "Used by Y" comment | Ties code to vanished context; rots | Delete — belongs in the commit message |
| Parallel near-identical functions | `handle_red/blue/green` differing in one literal | Parameterize |
| Premature abstraction | Helper with one caller, "for future reuse" | Inline until a second caller actually appears |

## Delete on Sight — Your Own Responses

| Pattern | Fix |
|---|---|
| Restating the user's question | Start with the answer |
| "Great question!", "Certainly!", "Of course!" | Delete |
| Summarizing what you just did when the diff is visible | Delete |
| Offering unrequested follow-ups | Delete |
| Explaining what a tool call will do after already calling it | Delete |
| Narrating internal deliberation | Delete — state the decision, not the process |

## The One-Comment Rule

Default: zero comments. Write one **only** when the *why* is non-obvious — a hidden constraint, a subtle invariant, a workaround for a specific bug, behavior that would surprise a reader.

## Rationalizations — None of These Work

| Excuse | Reality |
|---|---|
| "It flows better" | Flow comes from less text, not more. |
| "It is clearer with the extra sentence" | If the claim is unclear, fix the claim. Do not add a second sentence. |
| "It is more polite" | Politeness is not information. Delete. |
| "Future readers might not know" | Future readers can read the code. Comments rot; code does not. |
| "Defensive coding is good practice" | Defensive code at internal boundaries hides bugs. Trust your own functions. |
| "The summary helps skimmers" | A good document is skimmable without a summary. |
| "I want to acknowledge what they said" | Answering is acknowledgment. |
