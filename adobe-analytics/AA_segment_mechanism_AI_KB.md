# Adobe Analytics segmentation — machine-oriented knowledge base

**Purpose:** Encode `AA_segment_mechanism.html` (human-oriented) as structured facts for LLM retrieval. **Language:** English (canonical). **Human-readable Japanese source:** same folder, `AA_segment_mechanism.html`.

**Product:** Adobe Analytics (AA). Customer Journey Analytics (CJA) uses Person / Session / Event; mechanics align.

---

## 0. Canonical terminology

| Key | AA term | CJA equivalent | ID notation in examples |
|-----|---------|----------------|-------------------------|
| `visitor` | 訪問者 | Person | `P1`, `P2`, … |
| `visit` | 訪問 | Session | `S1`, `S2`, … |
| `hit` | ヒット | Event | `E1`, `E2`, … |

**Invariants**

- `hit.visit_id` is shared by all hits in one visit.
- `hit.visitor_id` is shared by all hits for one visitor (across visits).

---

## 1. Mental model: segment as identifier selection

**Definition:** A segment **selects rows** in reporting by **identifiers** (hit ID, visit ID, or visitor ID). “Match” means: include **all** rows whose identifier equals the identifier(s) the segment resolved to, at the **return scope** of the outermost container.

---

## 2. Containers: evaluation scope equals return scope

**Invariant:** For a non-sequential container, **evaluation_scope ≡ return_scope** (always the same).

| Container | Evaluates at | On match, returns |
|-----------|----------------|-------------------|
| Hit | Each hit | Matching hits only |
| Visit | Each visit (all hits in visit considered) | **All hits** in matching visits |
| Visitor | Each visitor (all hits considered) | **All hits** for matching visitors |

**Consequence:** Visit / Visitor containers often return hits that **did not** satisfy the positive condition (e.g. cart after product in a `[Visit: page=product]` segment), because the **unit of return** is the whole visit or visitor.

---

## 3. AND / OR and nesting

### 3.1 Single container

- **Hit + AND:** Conditions apply to the **same** hit. Example: `page=home AND page=product` on one hit → always false (one page value per hit in normal page dimension).
- **Visit + AND:** Conditions apply within the **same visit** but may be on **different** hits. Example: `page=home AND page=product` in one visit → true if both pages appear somewhere in that visit.

### 3.2 Nested containers (AND)

- A child container evaluates to a **boolean** for the parent: “does a sub-scope exist where this is true?”
- `[Visitor: [Visit: cart] AND [Visit: checkout]]` → cart and checkout may be in **different** visits; both must exist somewhere on the visitor timeline.

### 3.3 Outer scope wins for return set

- **Outermost container** sets **what rows are returned**.
- Inner containers / metrics contribute **truth values only**, unless structure implies otherwise.
- Example: `[Visit: page=product AND [Visitor: Orders > 0]]` returns **only hits in visits** that have product **and** where the visitor has lifetime orders > 0. It does **not** return the visitor’s entire history unless outer scope is Visitor.

### 3.4 Inverted nesting (Hit outside, Visitor inside)

- Still: outer = return scope, inner = predicate.
- `[Hit: page=cart AND [Visitor: Orders>0]]` → only **cart** hits that belong to visitors with orders.
- `[Visitor: page=cart AND Orders>0]` → **all** hits for visitors who have both a cart hit and orders (different shape).

---

## 4. Simple negation (`!=`, does not exist, etc.) is NOT “exclude whole visit”

**Critical semantics:** In Visit / Visitor containers, negation is evaluated as **existence of a hit matching the negated predicate inside the scope**, not as “no hit matches the positive form.”

**Example:** `[Visit: page != plp]`

- Meaning used by the product: “There exists at least one hit in this visit where page is not plp.”
- Therefore `home → plp → product` **matches** (home is a non-plp hit) even if the analyst meant “visit never touched plp.”

| Container | `page != plp` meaning |
|-----------|------------------------|
| Hit | This hit’s page is not plp (intuitive) |
| Visit | Visit contains **some** hit with page ≠ plp |
| Visitor | Visitor timeline contains **some** hit with page ≠ plp |

**To express “visit never contains plp”** use an **Exclude Visit** (or equivalent visit-level exclusion), not bare `!=` on Visit.

---

## 5. Exclude **containers** (Hit / Visit / Visitor) — scope alignment

**Concept:** Exclude containers are **not** “delete rows from the table.” They inject **negation at a chosen grain** (hit, visit, or visitor) into the segment logic.

| Exclude container | Negation unit | If inner condition matches… |
|-------------------|---------------|-----------------------------|
| Exclude Hit | Hit | That **hit** is excluded / fails at hit grain |
| Exclude Visit | Visit | The **entire visit** fails the parent (visit excluded) |
| Exclude Visitor | Visitor | The **entire visitor** fails the parent |

### 5.1 High-error pattern: Exclude Hit under Visit + AND

**Requirement (intent):** Visits that contain **home** and **never** contain **plp**.

**Wrong pattern A — negation:**

```text
[Visit: page=home AND page != plp]
```

- Evaluates as: exists a hit with home **and** exists a hit with not-plp.
- A `home` hit is always “not plp” on that hit → **equivalent** to `[Visit: page=home]`.
- Visits with `home → plp → …` still pass.

**Wrong pattern B — Exclude Hit nested in Visit:**

```text
[Visit: page=home AND [Exclude Hit: page=plp]]
```

- `Exclude Hit` marks plp hits as excluded at **hit** level; AND is resolved by finding hits that satisfy **both** sides at **hit** level where the engine combines them.
- The `home` hit passes `[Exclude Hit: plp]` (it is not plp). So again **equivalent** to `[Visit: page=home]`.
- **Exclusion effect on “any plp in visit” is lost.**

**Correct pattern C — Exclude Visit:**

```text
[Visit: page=home AND [Exclude Visit: page=plp]]
```

**Evaluation steps (visit grain):**

1. Visit contains a `home` hit? → must be true.
2. Visit contains **any** `plp` hit? → if true, **Exclude Visit** fails the visit (entire visit out).
3. Return visits where (1) true and (2) false — i.e. home **and** no plp in that visit.

**Invariant for analysts:** To exclude “anything about a **visit**,” the Exclude container’s **negation unit** should be **Visit** (or higher if intent is visitor-wide), not Hit, when the parent logic is visit-scoped AND that must reason about **whole-visit** presence/absence.

### 5.2 Recipe table (from source doc)

| Goal | Wrong | Right |
|------|---------|-------|
| Visits with home, **no** plp in same visit | `[Visit: home AND [Exclude Hit: plp]]` or `page!=plp` tricks | `[Visit: home AND [Exclude Visit: plp]]` |
| Purchaser visitors but **drop purchase visits** | `[Visitor: [Exclude Hit: order]]` | `[Visitor: [Visit: exists] AND [Exclude Visit: order]]` (pattern: ensure visit-level semantics; adjust “exists” to your UI primitive) |
| Remove **entire** visits that ever hit error page | `[Visit: [Exclude Hit: error_page]]` | Top-level `[Exclude Visit: error_page]` (or equivalent top-level exclude visit) |

**Note:** Exact builder syntax for “visit exists” may vary; the **logic** is: visitor-level return with visit-level exclusion requires **visit-grained** exclude, not hit-grained.

---

## 6. Sequential segments — components

**Orthogonal axes:**

1. **Sequence scope:** Visit vs Visitor (whether A THEN B must occur in one visit or may span visits).
2. **THEN chain:** ordered checkpoints (each checkpoint can be INCLUDE or EXCLUDE type in the sequential UI).
3. **After / Within:** proximity window between checkpoints (clock often anchored “after leaving” prior checkpoint per Adobe docs cited in source).
4. **Return mode:** Include Everyone vs Only After Sequence vs Only Before Sequence (and AND of two sequential groups for interval extraction).

---

## 7. Sequential segments — five Adobe principles (webinar)

Source: [Mastering Sequential Logic: Starts and Stops](https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/sequential-logic-start-stop)

1. All **INCLUDE** checkpoints in the THEN chain must be encountered **in order** before matching proceeds as intended.
2. Matching continues until an **EXCLUDE** checkpoint fires, then behavior depends on mode (see §9).
3. Multiple valid sequences **each** contribute (greedy / maximize returned data — see source examples).
4. Use **Within / After** to force required proximity.
5. Validate new definitions against real data.

**Scan model (THEN):** Scan hit stream in time order; find A; from pass point search for B; repeat. **Greedy:** when multiple completions exist, AA picks the outcome aligned with “maximum data” behavior illustrated in the HTML doc.

---

## 8. After / Within (combined)

**After:** B is not sought until minimum time/hits/visits after leaving prior checkpoint (per Adobe sequential-segment doc).

**Within:** B must appear within maximum window after that anchor; otherwise sequence fails.

**After + Within together:** Both constraints apply **in parallel from the same anchor** (Adobe doc: not sequential composition of windows).

Example semantic: `[After 1 day, Within 30 days]` between A and B → B must fall in **[+1d, +30d]** from anchor (see source for exact clock semantics).

---

## 9. Return modes vs last/first INCLUDE

**Only After Sequence**

- Collection start = hit matching the **last INCLUDE** checkpoint in the satisfied THEN chain.
- **EXCLUDE checkpoints do not** define the collection start anchor.
- Quote (webinar): when all THEN **include** checkpoints are met, matching of data starts from there.

**Only Before Sequence**

- Think **reverse** scan from end of scope.
- Collection end boundary = **first INCLUDE** checkpoint in the definition (inclusive of that hit per source examples).
- Greedy edge cases: when multiple A→B completions exist, which A is “first INCLUDE” for boundary uses the **greedy** pairing shown in the HTML (last A in the greedy pairing for the illustrated P2 case).

**Include Everyone**

- Sequence is **qualifying**; return **all** data in the outer container scope when qualified.

---

## 10. Sequential **EXCLUDE checkpoints** (not the same as Exclude Visit container)

These are **checkpoints inside a THEN sequence**, not the “Exclude Hit/Visit/Visitor” **container** wrappers from §5. Naming in English materials: **EXCLUDE checkpoint** vs **exclude container**.

### 10.1 Include Everyone (default) — role by position

| Position | Pattern | Role of EXCLUDE |
|----------|---------|------------------|
| First | `(EXCLUDE A) THEN B` | **Qualifying absence:** no A before B on the path; if A appears before B → **no match, nothing returned** for that visitor/visit scope |
| Middle | `A THEN (EXCLUDE B) THEN C` | **Qualifying absence between:** after A, before C, **no** B |
| Last | `A THEN B THEN (EXCLUDE C)` | **Qualifying absence after B:** after B, **no** C anywhere later in scope |

### 10.2 Only After + **last** EXCLUDE checkpoint

- Same **syntax position** as “qualifying no C after B” under Include Everyone, but with **Only After**, the last EXCLUDE often acts as **forward collection terminator**: keep collecting after last INCLUDE until C would fire — then **stop** (C excluded from return). Sequence still **matches**; data is **trimmed**.
- Webinar quote pattern: continues in Only After / Only Before direction until exclude condition seen.

### 10.3 Only Before + **first** EXCLUDE checkpoint

- Mirror of §10.2 going backward: first EXCLUDE can act as **backward terminator** while sequence still matches; returns data **between** C and A as illustrated in source (7-5).

### 10.4 Mode flip warning (highest confusion)

**Same** sequential shape `(EXCLUDE C) THEN A`:

- **Include Everyone:** if C ever appears before A as defined → **fail** (no data).
- **Only Before:** C may act as **boundary** → data **between** C and A can return.

**Invariant for LLMs:** For sequential EXCLUDE, **always resolve:** (a) Include Everyone vs Only After vs Only Before, (b) position of EXCLUDE in THEN chain, (c) Visit vs Visitor sequence scope — before predicting rows.

### 10.5 Summary matrix (from source; “unknown” = no clear official source in doc)

| EXCLUDE position | Include Everyone | Only After | Only Before |
|------------------|------------------|------------|---------------|
| First `(EXCL A) THEN B` | Qualifying: no A before B | Unknown in source | **Backward terminator** (documented pattern 7-5) |
| Middle `A THEN (EXCL B) THEN C` | Qualifying: no B between A and C | Unknown | Unknown |
| Last `A THEN B THEN (EXCL C)` | Qualifying: no C after B | **Forward terminator** (documented pattern 7-4) | Unknown |

**Operational rule:** Cells marked **Unknown** require **empirical validation** in workspace or doc update — do not infer from analogy alone.

### 10.6 Minimum checkpoints

Adobe: only **two** checkpoints required to use THEN; one may be EXCLUDE. Valid minimal chains like `A THEN (EXCLUDE C)` or `(EXCLUDE C) THEN A` exist; behavior still depends on **mode** (§10.4).

**“At a minimum…” note from webinar (source):** With Only After + terminal EXCLUDE (or Only Before + leading EXCLUDE), if INCLUDE matched, **at least** the INCLUDE checkpoint hit is returned; trimming still yields non-empty match when sequence qualified.

---

## 11. Practical composite: Visitor + Visit predicate + Exclude Visit

**Pattern:**

```text
[Visitor:
    [Visit: page=product]
    AND
    [Exclude Visit: event=purchase]
]
```

**Semantics:**

- Visitor must have **some** visit with product.
- Visitor must **not** have **any** visit that contains purchase (exclude visit grain).
- Returns **all hits** for qualifying visitors (outer Visitor return scope).

---

## 12. Reading procedure (composite segments)

1. Identify **outermost container** → sets **return grain** (hit / visit / visitor).
2. Evaluate nested pieces **inner → outer** as booleans or row filters per AA rules.
3. Where **Exclude *container*** appears, apply negation at **that** grain immediately when reasoning.
4. If **sequential**: run **scan model** for THEN; apply **After/Within**; then apply **return mode**; then resolve **EXCLUDE checkpoint** role using §10 matrix + mode.
5. If multiple completions: apply **greedy** examples from source before finalizing row set.

---

## 13. Design checklist (exclude-heavy)

- What grain must be absent: **hit**, **visit**, or **visitor**?
- Does **Exclude *container*** grain match that intent?
- Under **Visit** parent, is there an **Exclude Hit** paired with AND intended to ban **any** occurrence in the visit? → likely **wrong**; upgrade to **Exclude Visit** (or restructure).
- For **sequential** EXCLUDE: confirm **Include Everyone vs Only After vs Only Before** before interpreting EXCLUDE as “qualifying” vs “terminator.”

---

## 14. Official references (URLs)

- Sequential segments (AA): https://experienceleague.adobe.com/en/docs/analytics/components/segmentation/segmentation-workflow/seg-sequential-build
- Sequential segments (CJA): https://experienceleague.adobe.com/en/docs/analytics-platform/using/cja-components/segments/seg-sequential-build
- Webinar — Starts and Stops: https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/sequential-logic-start-stop
- Webinar — Visual framework: https://experienceleague.adobe.com/en/docs/events/adobe-customer-success-webinar-recordings/2025/multisolution2025/mastering-sequential-logic
- Excluding between checkpoints: https://experienceleague.adobe.com/en/docs/events/the-skill-exchange-recordings/analytics/may2022/tips-and-tricks

---

## 15. Epistemic status / gaps (from source plan + HTML)

- **Exclude Hit inside Visit** under arbitrary AND/OR trees: high-level pitfall documented; **not** claimed to be exhaustive formal spec from Adobe.
- **After/Within** clock anchor details: follow current ExL “leave page A” wording; edge cases should be validated in UI.
- **Only After + middle EXCLUDE** and **Only Before + middle EXCLUDE**: marked **unknown** in source summary table — treat as **unverified** without testing.

---

**End of knowledge base.** Pair with `AA_segment_mechanism.html` for narrative, diagrams, and hitstream examples.
