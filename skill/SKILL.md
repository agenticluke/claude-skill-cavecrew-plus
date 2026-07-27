---
name: cavecrew
description: >
  Decision guide for delegating code tasks to caveman-style subagents. Use
  `cavecrew-investigator` to locate code, `cavecrew-builder` for clear edits
  limited to 1-2 files, and `cavecrew-reviewer` to inspect diffs for bugs.
  Choose Cavecrew when compressed, structured results are more useful than
  explanatory prose. Trigger phrases include "delegate to subagent", "use
  cavecrew", "spawn investigator", "spawn builder", "spawn reviewer", "save
  context", and "compressed agent output".
---

# Cavecrew

Cavecrew provides three subagent presets that return terse, structured results. They cover the same general roles as vanilla exploration, editing, and review agents, but use fewer tokens in the main thread.

Use Cavecrew when the task is bounded and the main thread needs facts or edits, not extended explanation.

## Choose the right agent

| Task | Use |
|---|---|
| Find where a symbol is defined, called, tested, or referenced | `cavecrew-investigator` |
| Explore architecture, explain behavior, or suggest approaches | Vanilla `Explore` |
| Make an obvious, surgical edit affecting no more than 2 files | `cavecrew-builder` |
| Implement a new feature, change 3 or more files, or perform a cross-cutting refactor | Main thread or `feature-dev:code-architect` |
| Review a diff, branch, commit, or specific file for defects | `cavecrew-reviewer` |
| Provide a deep review with rationale, tradeoffs, and alternatives | Vanilla `Code Reviewer` |
| Answer a trivial question already resolved in the main thread | Main thread |

Rule of thumb: choose Cavecrew when a structured answer at roughly one-third the usual length is sufficient. Choose a vanilla agent when reasoning or prose is part of the deliverable.

Do not delegate when setup and context transfer would cost more than completing the task inline.

## Agent contracts

### `cavecrew-investigator`

Use for read-only code discovery. Provide a concrete search target and, when known, a directory or file scope.

Expected output:

```text
<Header>:
- path:line — `symbol` — short note
totals: <counts>.
```

If nothing is found:

```text
No match.
```

The result must:

- Put the file path first.
- Include a line number for every finding.
- Wrap symbols in backticks.
- Separate distinct definitions, callers, references, and tests when requested.
- Report generated, vendored, fixture, or duplicate matches as such.
- Say `No match.` only after searching the requested scope and obvious naming variants.
- Never modify files.

### `cavecrew-builder`

Use only when the target files and desired change are already clear. Limit the task to 1-2 files, including tests, snapshots, generated files, and configuration files.

Expected output:

```text
<path:line-range> — <change in 10 words or fewer>.
verified: <check performed and result | mismatch @ path:line>.
```

If the task cannot be completed safely, the first token must be one of:

```text
too-big.
needs-confirm.
ambiguous.
regressed.
```

Meanings:

- `too-big.`: completion requires more than 2 files or a broader refactor.
- `needs-confirm.`: the task requires an irreversible, destructive, privileged, or externally visible action.
- `ambiguous.`: multiple materially different edits satisfy the request.
- `regressed.`: verification failed or the change introduced a new failure.

The builder must:

- Preserve unrelated user changes.
- Stop rather than expanding scope beyond 2 files.
- Re-read every edited region.
- Run the narrowest relevant available check.
- Report a missing or unavailable verification command explicitly.
- Never claim verification from re-reading alone when tests, linting, type-checking, or compilation were requested.
- Avoid destructive operations and external side effects unless explicitly authorized.

If the requested code change affects one source file but requires a third file for tests or generated output, return `too-big.` rather than silently omitting required work.

### `cavecrew-reviewer`

Use for defect-focused review of a defined diff, branch, commit range, or file set.

Expected output:

```text
path:line: <emoji> <severity>: <problem>. <specific fix>.
totals: N🔴 N🟡 N🔵 N❓
```

If no actionable defects are found:

```text
No issues.
```

Severity markers:

- 🔴: likely correctness, security, data-loss, or production failure.
- 🟡: meaningful bug, edge-case failure, or maintainability risk.
- 🔵: minor defect or low-risk improvement.
- ❓: issue depends on missing context or an unverified assumption.

The reviewer must:

- Review only the requested scope.
- Prefer actionable defects over style preferences.
- Include exact paths and line numbers.
- Sort findings by file, then line number.
- Distinguish confirmed defects from assumptions using ❓.
- Note when tests do not cover a changed behavior.
- Avoid architecture essays and general praise.
- Use `No issues.` only when no actionable finding remains.

## Common workflows

### Locate, fix, verify

1. Ask `cavecrew-investigator` for definitions, callers, and relevant tests.
2. Select a change that fits within 1-2 files.
3. Give `cavecrew-builder` exact paths, locations, desired behavior, and verification criteria.
4. Ask `cavecrew-reviewer` to inspect the resulting diff.

Do not continue to the builder if the investigation reveals a cross-cutting change.

### Parallel investigation

Use 2-3 investigators only when the searches are independent, such as:

- Definitions and implementations.
- Callers and data flow.
- Tests and fixtures.

Give each investigator a distinct scope. Do not launch duplicate searches, and do not parallelize when one result is required to define the next search.

### Direct edit

If the exact site and behavior are already known, skip investigation and call the builder directly.

## Concrete example

Request:

```text
The CLI crashes when config.toml omits timeout. Use cavecrew and keep the fix small.
```

Delegation:

```text
cavecrew-investigator:
Find where `timeout` is parsed and consumed, plus the closest tests.
Scope: src/ and tests/.
Return definitions, callers, and tests only.
```

Possible result:

```text
Timeout flow:
- src/config.py:41 — `load_config` — reads required `timeout` key
- src/client.py:18 — `Client.__init__` — receives parsed timeout
- tests/test_config.py:27 — `test_load_config` — covers explicit timeout only
totals: 1 definition, 1 caller, 1 test.
```

Builder request:

```text
cavecrew-builder:
Edit src/config.py and tests/test_config.py only.
Default a missing timeout to 30 without changing explicit values.
Add a regression test. Run the narrow config test file.
```

Possible result:

```text
src/config.py:41-43 — default missing timeout to 30.
tests/test_config.py:39-48 — cover missing timeout.
verified: tests/test_config.py passed.
```

Reviewer request:

```text
cavecrew-reviewer:
Review the current diff in src/config.py and tests/test_config.py for correctness,
backward compatibility, and missing edge cases.
```

## Avoid these mistakes

- Do not use `cavecrew-builder` before identifying the files and intended behavior.
- Do not send a 3-or-more-file change to the builder.
- Do not hide required tests, snapshots, migrations, or generated output to force a task under the 2-file limit.
- Do not ask the reviewer for general feedback, architecture advice, or a prose summary.
- Do not ask the investigator to edit code.
- Do not treat `No match.` as proof that a symbol does not exist outside the searched scope.
- Do not expect Cavecrew output to be polished for end users. Paraphrase it when presenting results to a human.
- Do not repeat sensitive values, credentials, tokens, or private data in prompts or returned summaries.

## Auto-clarity

Subagents must switch from caveman-style fragments to clear English for:

- Security warnings.
- Destructive or irreversible-action confirmations.
- Permission or authorization requests.
- Sensitive-data handling.
- Any statement where terse wording could change the meaning.

Resume compressed output after the safety-critical message.