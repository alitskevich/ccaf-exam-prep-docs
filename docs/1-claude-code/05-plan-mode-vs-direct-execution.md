# Plan mode vs direct execution

Choose the right execution mode based on task complexity *before* writing any code.

| Mode | Use when | Examples |
|---|---|---|
| **Plan mode** | Complex, architectural, multi-file, multiple valid approaches | Monolith → microservices (45+ files), library migrations |
| **Direct execution** | Well-scoped, single-file, clear change | Single bug fix with stack trace, adding one validation check |

- **Hybrid**: plan mode for investigation, then direct execution for implementation — the plan becomes the spec.
- If complexity is stated upfront, enter plan mode **before** any code changes.
- **Heuristic:** if you'd ask a senior engineer to whiteboard before coding, use plan mode. If a junior could fix it from the stack trace, use direct execution.
- For multi-phase plan-mode work, pair `--plan` with `--use-explore` so verbose discovery stays in the [Explore subagent](03-extending.md#worked-example-the-explore-subagent) and the main session preserves context for design decisions.
- "Plan" doesn't mean "perfect plan"; you still iterate, but with a map.
- "Plan mode is slow" is the common objection; plan mode on a 45-file refactor is *much* faster than 2× rework.
- Treat the rule as a default, not a ceiling: 10+ files OR multiple valid architectural approaches → plan first.

```bash
# Plan-then-execute — discovery in planning mode, implementation direct
claude --plan "Investigate the slow checkout flow"
claude "Now implement the cache fix described in CheckoutService"

# Explore subagent prevents context exhaustion in multi-phase tasks
claude --plan --use-explore "Trace every reader of the orders table"
```

### Default workflow: explore → plan → code → commit

1. **Explore** — read the relevant code, map dependencies, surface constraints
2. **Plan** — design the change in plan mode, get alignment on the approach
3. **Code** — switch to direct execution; the plan is now the spec
4. **Commit** — verify against tests/screenshots, then commit

Skip planning for trivial edits (single-file fixes with a clear stack trace, adding a guard, debugging logs). Default to plan mode for tasks involving 10+ files or multiple valid architectural approaches.

A plan that only lists files-to-edit isn't a plan — it must surface *constraints* (cycles, shared state, contracts, sequencing).

### Anti-patterns

- Starting in direct execution and switching to plan mode only when complexity is discovered — constraints are found after changes are made, requiring rework ❌
- Writing comprehensive upfront instructions without codebase exploration — assumes knowledge that exploration would have provided ❌
- Direct on a 38-file refactor surfaces dependency surprises *after* edits — burning rework time ❌
- Plan mode on a one-line null check wastes a turn — pick the lighter tool ❌

### Use case: plan or direct?

For each task, decide: plan mode or direct execution?

1. Fix a null pointer exception; stack trace points to line 47 of `UserService.ts`.
2. Migrate from Axios to the native Fetch API across 38 service files.
3. Add a missing null check before an API call in `auth.service.ts`.
4. Restructure the authentication module to support OAuth 2.0 and SAML simultaneously.
5. Add a `console.log` for debugging in `payment.controller.ts`.

```bash
# Direct — narrow, low risk (1, 3, 5)
claude "Fix NPE at UserService.ts:47"
claude "Add null check before fetchUser() in auth.service.ts"
claude "Add console.log of incoming payload in payment.controller.ts"

# Plan first — multi-file or architectural (2, 4)
claude --plan "Migrate Axios → fetch across all 38 service files"
claude --plan "Refactor auth module to support OAuth2 + SAML side-by-side"
```

**Answers:** 1 → direct · 2 → plan · 3 → direct · 4 → plan · 5 → direct.

#### Worked example: skipping plan mode on a 45-file restructure

**Situation:** A monolith-to-microservices restructuring involving 45+ files starts in direct execution. Mid-implementation, undiscovered dependency constraints require significant rework of already-completed service boundaries.

**Root cause:** Complexity was known upfront (multi-file, architectural, multiple valid approaches) but plan mode wasn't used. Direct execution on architectural tasks surfaces constraints only *after* code changes are made.

**Fix:** Enter plan mode before any code changes — explore the codebase, map dependencies, design service boundaries, then implement.

```bash
# What went wrong
claude "Split monolith into auth/billing/notifications services"
# → Claude edits 20 files, discovers shared session store, rewrites 12, hits ORM cycle, rewrites 8 more

# What to do instead
claude --plan "Split monolith into auth/billing/notifications services"
# Plan-mode output:
#   1. Map cross-module imports (Grep)
#   2. Identify shared state (session store, ORM cycles, event bus)
#   3. Propose service boundaries; flag the 3 cycles that block clean split
#   4. Sequence: extract notifications first (zero inbound deps),
#      then billing, finally auth (highest fan-in)
#   5. Implement only after sequence approved
```
