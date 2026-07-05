# Validation, Reporting, and Verification

### Phase 3: Validate findings

Collect all findings from Phase 2 agents and **consolidate duplicates first**. Phase 2 deliberately overlaps agent scopes, so the same issue is frequently reported by more than one hunter — merge findings that share a root cause before validating, or you'll validate and report the same bug multiple times. For each remaining finding, launch a **separate `research` validation agent** that tries to disprove it. The hunting agents are biased toward finding things; the validation agents are biased toward killing false positives. This adversarial step is critical.

For findings from the same attack surface, batch them into one validation agent. Launch validation agents in parallel where they cover independent areas.

Each validation agent prompt should:

1. State the specific finding being validated (title, claimed attack, claimed impact)
2. Ask the agent to read the exact code paths and verify each step of the trace
3. Ask it to apply these tests:

**Validation tests:**

1. **Exploitation test**: Read the actual code at each step of the trace. Does the data flow work as claimed? Can you construct the exact input (HTTP request, CLI invocation, API call, crafted file, etc.) that triggers this?
2. **Impact test**: What does the attacker actually get? If the answer is "they learn field names" or "they cause an error", that's LOW at best.
3. **Baseline test**: Does the identified comparable have the same pattern? If yes, has it been exploited? If never exploited in years of production use, understand why before reporting.
4. **Mitigation test**: Is there another layer that prevents exploitation? Check middleware, database constraints, framework defaults.
5. **Parser/runtime behavior test**: If the exploit depends on how a parser or runtime handles specific input, verify against the actual spec or implementation — do not reason from intuition.

Tell each validation agent:

```
Your job is to DISPROVE this finding. Read the actual source code at every step. If you cannot disprove it, confirm it with the exact code that makes it exploitable. Return one of:
- "CONFIRMED: [explanation of why it's real, with code evidence]"
- "REJECTED: [explanation of what the finding got wrong, with code evidence]"

<repo intelligence tools instructions>
```

**Kill false positives aggressively, but don't kill real findings.** A short report with 3 real findings is worth more than a long report with 30 theoretical ones. An honest "nothing found" is valid — but push hard before reaching that conclusion.

### Phase 4: Report

Write the report to the output directory established in Setup.

**Output files:**

1. `REPORT.md` -- Main report with:
   - One-paragraph executive summary (honest assessment of security posture)
   - Identified baseline and how this application compares
   - Findings table (severity, title, one-line description)
   - Each finding with: file path, concrete attack scenario, impact, recommended fix
   - Hardening notes section (defense-in-depth suggestions, NOT findings)
   - Positive patterns section (what the codebase does well -- this calibrates trust in the audit)

2. `FINDINGS-DETAIL.md` -- For each finding rated MEDIUM or above:
   - Complete data flow from input to sink with file:line references
   - Exact HTTP request(s) to trigger
   - What the attacker gets
   - How the baseline comparable handles the same scenario

Keep it short. If the report is longer than the codebase deserves, you're padding.

### Phase 5: Structured output and schema check

For every finding that survived Phase 3 validation, produce a structured JSON object conforming to the schema defined in `report-schema.json` (in the same directory as this skill file — read it via the Read tool before writing output). Write the result to `<output-dir>/findings.json`.

The schema supports two verdict types via `oneOf`:

- **`confirmed`** — a validated vulnerability with full trace, execution, and remediation
- **`rejected`** — a finding that was investigated and determined to be factually incorrect

**Before writing `findings.json`:**

1. Read `report-schema.json` from this skill's directory. Follow it exactly — `additionalProperties: false` is enforced, so extra fields will make the output invalid.
2. For each finding, populate every required field. If you cannot fill `trace` with real file paths and line numbers verified against the source, the finding is not sufficiently verified — go back and verify it or reject it.
3. Run `node <skill-dir>/validate-findings.cjs <output-dir>/findings.json` to validate. It checks required fields, enum values, structural constraints, and `additionalProperties`. This is a structural check only — it confirms the JSON conforms to the schema, not that the findings are correct. Factual verification is Phase 6's job. Fix any failures before proceeding.
