# Dev Guardrails — Regression Prevention

> **Last updated:** 2026-04-29 · **Reconsider by:** 2026-07-29 · **Confidence:** high — written after the ManageTenants pagination regression (2026-04-28) and the AcceptInvitation blank-screen fix. Grounded in two real incidents, not theory.
>
> Practical process for preventing regressions when shipping via Lovable or Claude Code. Applies to optimisations, refactors, and data-fetch changes in particular — the category most likely to silently break other UI features.

---

## Why this exists

The ManageTenants pagination fix (April 2026) introduced a silent regression: the four summary stat cards (Total Clients, Active, Suspended, Closed) derived their counts from `tenants.length` — the in-memory array — rather than a dedicated database count query. When the fetch was capped at 100 rows, the cards showed 100 regardless of the real tenant count. No data was lost; the display was wrong. It was caught by a stakeholder, not by the team.

The root cause was not carelessness. It was that **Lovable does what you ask, nothing more** — paginate the fetch, and it paginates the fetch. It does not know what else depends on the array size unless you tell it. And there was no step in the shipping process that would have caught the regression before it hit production.

These guardrails close that gap.

---

## The three mandatory steps

### Step 1 — Blast-radius check (before writing the Lovable prompt)

Before sending any optimisation or data-fetch change to Lovable, ask Claude:

> "What else in this file or the codebase reads from or depends on the data this change touches? List every consumer."

Do this for:
- Any change to how data is fetched (adding `.range()`, switching to React Query, changing a hook)
- Any change to shared state (removing a `useState`, renaming a variable, merging two arrays)
- Any change to a component that other components import from

Record the answer. If there are consumers beyond the thing you're fixing, the Lovable prompt must account for all of them.

---

### Step 2 — Write a scoped Lovable prompt

Every prompt for a performance or data-fetch change must include two things the basic prompt omits:

**a) An explicit "do not touch" list**

Name every UI element that must produce the same result after the change:

> "Do not change how the stat cards, CSC load bar, filter counts, or pagination controls are computed. Only change [the specific thing]."

**b) A dependency note**

If the blast-radius check found consumers, state them explicitly:

> "Note: the Total Clients stat card reads from `tenants.length`. If you are changing how `tenants` is populated or sized, the stat card must be updated to use a separate count source."

Without this, Lovable will not know the dependency exists.

---

### Step 3 — Post-pull verification checklist

After every `git pull` of a Lovable change, run the relevant page checklist below before marking the task done. This is the minimum browser test — not QA, just a smoke check of the highest-risk surfaces.

**Always check:**
- [ ] The thing that was changed works on the golden path
- [ ] At least one edge case for the changed feature (empty state, error state, boundary value)

**For data-fetch or pagination changes, also check:**
- [ ] Summary / stat cards show correct totals (match what you'd expect from the DB)
- [ ] Row counts in tables match stat card totals
- [ ] Filters still narrow results correctly
- [ ] Any "Load More" or pagination controls work and update counts
- [ ] CSC load bar totals (if on a page with CSC assignment) still sum correctly

**For React Query migration (replacing bare useEffect):**
- [ ] Data loads on first visit
- [ ] Data refreshes correctly after a mutation (add, edit, delete)
- [ ] Realtime updates still trigger the correct re-fetch or invalidation
- [ ] No duplicate fetch calls visible in the network tab

---

## Known regression traps

These are patterns that have caused or will cause silent regressions. Treat each as a rule.

### Trap 1 — Deriving counts from a paginated array

**Pattern:** `tenants.length`, `items.filter(...).length`, or any count computed from an in-memory array that is populated by a paginated query.

**Risk:** If you paginate the fetch, the count reflects only the loaded page. Stat cards, CSC load bars, and filter counters all break silently — no error, just wrong numbers.

**Rule:** Summary statistics (totals, status counts, load distributions) must always come from a dedicated count query, never from the length of a paginated list. Use `select('id, status')` with no `.range()` limit in a separate hook, or use Supabase's `{ count: 'exact', head: true }` option.

---

### Trap 2 — Removing a useState that other JSX reads

**Pattern:** Removing or renaming a state variable during a React Query migration without checking all the JSX that references it.

**Risk:** The component silently renders nothing, or a section disappears, if JSX still references the old variable name.

**Rule:** When removing state during a migration, grep the file for every reference to that variable name before sending the Lovable prompt. Name them in the prompt.

---

### Trap 3 — Realtime handler calling a full-fetch function

**Pattern:** `.channel()` subscription handler calls `fetchAll()` or `fetchTenants()` — a function that re-runs all queries — instead of a scoped invalidation.

**Risk:** Every realtime event triggers a full re-fetch waterfall, compounding load and causing UI flicker.

**Rule:** Realtime handlers must call `queryClient.invalidateQueries({ queryKey: [...] })` for the affected query only, not a function that re-fetches everything. The ManageTenants fix (Phase 1, April 2026) established this pattern.

---

### Trap 4 — setLoading(false) outside a finally block

**Pattern:** `setLoading(false)` or `setValidating(false)` appears only inside the `try` block or after an `await`, not in `finally`.

**Risk:** If the fetch throws or times out, the loading state never clears. The user sees a permanent spinner with no way out.

**Rule:** Every `setLoading(true)` must have a matching `setLoading(false)` inside a `finally` block. No exceptions. The AcceptInvitation diagnosis (April 2026) found this pattern in multiple pages — see the platform-wide audit results in `unicorn-audit/audit/2026-04-28-accept-invitation-blank-screen-fix.md`.

---

### Trap 5 — Error branch that returns null

**Pattern:** `if (!data) { return null; }` as the error or loading-failed branch.

**Risk:** The user sees a completely blank page. No heading, no explanation, no retry. Toast notifications auto-dismiss — after a few seconds there is nothing on screen and no way to proceed.

**Rule:** Every `if (!data)` branch that can be reached after a failed fetch must render an explicit error state: a heading, a description, and a retry action. `return null` is only acceptable for loading skeletons where a spinner is already rendered in a parent.

---

## Page-specific smoke-test reference

Quick checklist per page for the highest-risk pages in the platform.

### ManageTenants

- Total Clients count matches the real DB count (not the page size)
- Active / Suspended / Closed / Archived counts sum to Total Clients
- CSC load bar names and numbers match the Active count distribution
- Table row count in the current view matches expectations given active filters
- Load More adds rows and all four stat cards update
- Adding a tenant via AddTenantDialog increments Total Clients
- CSC quick-assign updates the CSC load bar without full page reload

### AcceptInvitation

- Valid token → spinner → form pre-populated with name and email from invitation
- Expired token → error heading + description + Try again button (not blank screen)
- Invalid / missing token → same error UI as above
- Try again button re-runs validation without page reload
- Successful signup → redirect to dashboard within 2 seconds

---

## When to escalate vs. proceed

If the blast-radius check turns up consumers you cannot account for in the prompt, **stop and flag to Carl** before prompting Lovable. Do not guess.

If a post-pull check reveals a broken surface, **do not mark the task done**. Raise a follow-up prompt to Lovable immediately, or document the regression in `unicorn-kb/reference/brainstorm-log.md` with a severity rating.

If a Lovable change touches RLS policies or database migrations, also apply the checklist in `pinned/conventions.md → New table checklist`. These guardrails do not replace that one — they sit alongside it.
