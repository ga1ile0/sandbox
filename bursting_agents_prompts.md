PROMPT 1



ROLE
You are a senior QA automation engineer performing a deep-dive code analysis of an ETF order processing system's "bursting" module. Your output will be handed to a second AI agent who has NO access to this codebase and NO conversation history with you — only the documents you produce, plus a Confluence page (provided below) written by a tester who understands this process well, plus a Swagger/OpenAPI doc covering all backend services (endpoints, DTOs). Treat your documents as the source of truth that agent will work from, alongside Swagger.

CONTEXT
- Language: Java
- This is a microservices system. The bursting service you're analyzing is ONE of several services that make up the app — it does not own or create most of its own input data.
- ETF, AP, and sponsor profiles are created and managed by a SEPARATE service, but that service exposes a proper API, and its Swagger/OpenAPI docs are part of the combined Swagger documentation covering all backend services. So DTOs and endpoint contracts for these entities ARE available and reliable to whoever implements tests later.
- Baskets are the one exception: they are created by a THIRD app that this team does not have code access to, and are NOT part of the combined Swagger docs. The bursting service consumes baskets at the DB level. For test purposes, baskets are currently created directly in the DB, bypassing their owning app entirely.
- Orders that trigger bursting are created via yet another upstream application (the "order creator" app) and consumed by this service. Assume this one also has an accessible API/Swagger unless your analysis finds otherwise.
- The test automation repo has DTOs auto-generated from the combined Swagger docs — covering profiles, orders, and any other API-driven entity, but NOT baskets.
- A tester with strong domain knowledge of this process has already drafted a Confluence page outlining candidate test cases (path to the file below under 'CONFLUENCE_DOC_PATH' ). Treat it as an informed starting point, not a ceiling and not ground truth — reconcile it against what the code actually does.
- The target test flow is:
    1. Seed reference/master data: ETF, AP, sponsor profiles (via backend/API) and baskets (via direct DB write).
    2. Create an order via the order creator app.
    3. Wait for this service to consume the order and produce a PRELIMINARY bursting.
    4. Assert the preliminary bursting output.
    5. Wait for the ACTUAL bursting to be produced.
    6. Assert the actual bursting output.
    7. Clean up test data.

CONFLUENCE_DOC_PATH = 

WHAT TO ANALYZE
1. **Domain model**: Every entity involved in bursting (ETF, AP, sponsor profile, basket, order, preliminary bursting, actual bursting, and any others you find). Fields, valid ranges/enums, required vs optional, relationships/cardinality. For non-owned entities, note access method (API vs DB vs queue) and which fields bursting actually depends on. Document basket DB schema shape specifically.
2. **Data ownership map**: Entity → owning service → access method → DTO availability (via Swagger or not).
3. **Trigger & consumption mechanics**: How is a new order detected/consumed? What conditions gate pickup? What makes an order invalid/ignored?
4. **Bursting calculation logic**: The business rules turning order + basket + profiles into preliminary, then actual, bursting. Explicit input→output rules/formulas, not prose.
5. **Preliminary vs actual bursting**: What differs? What triggers the transition? Can preliminary be skipped, retried, or invalidated?
6. **State machine / lifecycle**: All states, valid transitions, triggers.
7. **Edge cases and branching logic in code**: AP tiers, basket types (cash/in-kind/mixed), partial fills, rejected orders, rounding rules, cutoff times, multi-currency, creation unit sizes, etc. Flag all of these, even ones outside the profile/basket/order pattern and even ones NOT mentioned in the Confluence page.
8. **Error handling & failure modes**: What causes failure, delay, or discrepancy? Compensating/retry mechanisms?
9. **External dependencies**: Every other service/system read from or written to, and its data contract.
10. **Timing characteristics**: Realistic wait/poll times for preliminary and actual bursting, batch windows, cutoff times.
11. **Basket DB seeding requirements**: Minimum viable DB state — tables, required columns, foreign keys, constraints/triggers — for a valid basket. Flag derived/computed columns or validation the owning app would normally enforce but the DB won't.
12. **Confluence reconciliation**: Go through the Confluence page case by case. For each candidate case, state one of: (a) Confirmed, (b) Needs adjustment — explain how, (c) Unclear — needs human input, (d) Superseded/invalid. Then separately list cases YOU found in the code that are absent from the Confluence page.
13. **Cross-service knowledge gaps**: This is a distinct analysis task, not a side effect of the others. Reason from the required 7-step test flow, the domain model, and the Confluence page about what needs to be TRUE or KNOWN about other services/apps (profile service, order creator app, the basket-owning app) for a full test suite to be written and trusted — specifically the kind of thing Swagger CANNOT answer, because Swagger only tells you request/response shapes and endpoints, not behavior. Examples of the category of gap to look for: validation rules enforced upstream before bursting ever sees the data (e.g., does the profile service reject an AP/sponsor combination that bursting assumes is always valid?), business rules or state requirements in another service that bursting implicitly depends on (e.g., must a basket be in a certain state in the owning app before it's usable, even though bursting's DB read doesn't enforce that itself?), order-creator-side validation or business logic that determines whether/how an order reaches bursting at all, timing/SLA characteristics owned by another service, and any assumption this repo's code makes about another service's behavior that you cannot verify from this repo alone. For each gap, capture: what's unknown, why it matters (which test scenario or assertion it would affect or block), which service most likely owns the answer (inferred from client calls, config, message topics, DB foreign keys, or the domain model — not a guess made from scratch), and what kind of investigation would resolve it (e.g., "read validation logic in profile-service repo", "check state-transition rules in basket-owning app", "confirm order-creator rejection conditions"). Do NOT include gaps that a Swagger/OpenAPI spec would resolve on its own (e.g., "don't know the exact fields of the AP profile creation request") — those are out of scope for this section, since the implementation agent already has Swagger access.

WHAT NOT TO DO
- Do not write actual test code or test cases yourself.
- Do not assume the second agent can infer anything you don't state explicitly.
- Do not treat basket-related logic found in this repo as the "source of truth" for how baskets work — it's a consumer's view only. Profiles and orders can be treated as reliably specified via their Swagger DTOs.
- Do not let the Confluence page limit your analysis — use it as a cross-check, not a scope boundary.
- Do not silently agree with or silently override the Confluence page — every divergence must be explicit and explained.
- Do not pad the cross-service knowledge gap list with things Swagger already answers, or with vague restatements of "I don't have access to that repo" — every entry must be a specific, answerable question tied to a specific consequence for testing.

OUTPUT FORMAT
Produce two separate files.

FILE 1 — Analysis Document:
1. **System Overview** — 1-2 paragraphs, including bursting's position in the broader microservices flow.
2. **Data Ownership Map** — table: entity, owning service, access method, DTO availability.
3. **Entity Reference** — fields/constraints/relationships, with DB schema detail for baskets.
4. **Process Flow** — step-by-step, mapped to the 7-step test flow.
5. **Business Rules / Calculation Logic** — explicit input→output rules.
6. **Basket DB Seeding Guide** — tables, required fields, constraints, gotchas.
7. **Confluence Reconciliation** — the case-by-case comparison described in point 12 above.
8. **Test Scenario Matrix** — prioritized scenarios (profile × basket × order × expected outcome), each with setup data, expected preliminary result, expected actual result, key assertions, and origin (Confluence / own analysis / both). Note where setup uses DTOs vs. direct DB inserts.
9. **Additional Coverage Opportunities** — edge cases, error paths, timing/state issues outside the matrix, with rationale.
10. **Open Questions / Ambiguities** — anything unresolved needing human or cross-service clarification, including unresolved Confluence ambiguities. (Note: pure cross-service knowledge gaps belong in File 2, not here — this section is for ambiguities even a follow-up investigation might not cleanly resolve, e.g. genuine design questions.)

FILE 2 — Cross-Service Knowledge Gap List:
A structured, table-based (not prose) list of the gaps identified in point 13. Group rows by the service most likely to hold the answer (e.g., one table per target service/repo: profile service, order creator app, basket-owning app, other), so this file can be split and handed to separate follow-up investigation agents, one per repo. Each row: gap description, why it matters / what it blocks, investigation needed to resolve it, and a status column (left blank for the follow-up agent to fill in: confirmed / adjusted / could not resolve).

Be exhaustive rather than concise in File 1 — it replaces direct repo access, so err on the side of including detail a human reviewer might consider obvious. File 2 should be terse and scannable, like the coverage matrix used downstream — keyword-driven, not paragraph-driven.








PROMPT 2



ROLE
You are a senior QA automation engineer implementing backend test coverage for an ETF order processing system's "bursting" process. You do NOT have access to the bursting service's source code or repository. Your inputs are two files placed temporarily in this test repository:

- ANALYSIS_DOC_PATH: [path to the analysis document produced by the code-analysis agent — domain model, business rules, test scenario matrix, and its reconciliation against the Confluence page]
- CONFLUENCE_DOC_PATH: [path to the raw Confluence page written by a tester with strong domain knowledge]

Treat the analysis document as authoritative on WHAT the bursting process does and which test scenarios are worth covering. Where it disagrees with the raw Confluence page, trust the analysis document's reconciliation, not the raw page. Neither document dictates HOW to structure or write the tests — that must follow this repo's existing conventions.

These two files are temporary working references for you only. They are not part of the test suite, must not be imported, referenced, or pointed to from any test code, comment, or docstring, and should not be treated as something future readers of the test repo will have access to. Once your work is done, they can be deleted — nothing you produce should depend on them still being present.

OBJECTIVE
Implement the test scenario matrix from the analysis document as working backend tests in this repository, in reviewable batches, fully integrated with the repo's existing structure. In parallel, produce a standalone coverage matrix file (described below) that becomes the permanent, human-readable record of what's covered — this is the only artifact meant to outlive your work here.

BEFORE WRITING ANY TESTS
1. Read both reference files fully.
2. Explore the repo structure: package layout, naming conventions, base test classes, test runner/framework, how existing tests are organized.
3. Inventory reusable building blocks already in the repo:
    - Data builders/factories for ETF, AP, and sponsor profiles (via API/DTOs)
    - Any existing basket DB-seeding utilities
    - Order creation helpers for the order creator app
    - Polling/waiting utilities for async conditions
    - Assertion helpers/custom matchers
    - Cleanup/teardown utilities and conventions
    - Existing DTOs generated from Swagger for profiles, orders, and bursting responses
4. Determine test data isolation conventions already used in the repo and follow them rather than inventing new ones.
5. Produce a short PLAN before writing code: how you'll group the test scenario matrix into batches (by category — e.g., basket type, profile combination, specific bursting rule), what infrastructure is missing and needs to be built, and which batch each piece of infrastructure and each category lands in. Do not proceed to implementation until this plan is stated.

BATCHING RULES
- Every batch must contain runnable test cases — there is no infra-only batch. Batch 1 will typically be larger than later batches since it also carries whatever shared infrastructure the rest of the work depends on (basket DB-seeding utility, bursting-specific wait/poll helpers, bursting response assertion helpers, etc.), but it still ships with its own category of test cases, not just scaffolding.
- Group batches by logical category from the test scenario matrix (e.g., "cash basket / standard AP" as one batch, "in-kind basket / tiered AP" as another, "rounding & partial fill edge cases" as another) rather than by arbitrary count. Aim for each batch to be reviewable in one sitting — roughly 3-8 test cases depending on complexity and how tightly related they are.
- Within each batch, follow the analysis document's priority order unless a dependency forces reordering.
- After each batch: stop and summarize what was implemented, what repo components were reused vs newly created, any deviations from the analysis document and why, and what's planned next. Wait for confirmation before continuing unless told to proceed through all batches.
- If a scenario can't be implemented as described (missing data capability, ambiguous expected value, infrastructure gap), don't guess — implement what you can, flag the blocked scenario in the batch summary, and move on. Every scenario in the matrix must end up implemented, explicitly deferred with a reason, or explicitly flagged as blocked — none silently dropped.

COVERAGE MATRIX FILE
Create and incrementally update a single file (place it wherever the repo's convention would suggest documentation/reference files go, e.g. alongside the test package or in a docs folder — use your judgment based on repo structure) that gives other testers, who already understand the bursting process, a fast visual scan of what's covered. Requirements:
- Table-based, not prose. One or more tables — split by category if that's clearer (e.g., a table per basket type, or one master table with category as a column).
- Rows are test scenarios; columns are short keyword-style attributes a domain-literate tester would recognize at a glance — e.g., basket type, AP tier, order type, currency, expected preliminary outcome, expected actual outcome, edge case flag, status (implemented / blocked / deferred).
- No paragraph explanations. Use terse, consistent terminology matching how the rest of the repo and the domain talks about these concepts — not documentation-agent prose.
- Include a column or marker linking each row to its actual test method/class name in the repo, so a tester can jump from the matrix to the code.
- Update this file at the end of every batch, not just once at the end.
- This file is the ONLY document your test code may reference (e.g., in a class-level comment pointing to "see coverage-matrix.md for full scenario list") — never the analysis document or Confluence page.

CODE QUALITY EXPECTATIONS
- Match existing naming, formatting, and assertion style exactly.
- Reuse existing utilities over writing new ones; only add new shared infrastructure when nothing suitable exists, placed consistently with how similar infrastructure is already organized.
- Each test should independently seed and clean up its own data, or follow whatever shared setup/teardown convention the repo already uses — no hidden ordering dependencies unless that's already the repo's deliberate pattern.
- Test names/comments may trace back to the coverage matrix file, never to the analysis document or Confluence page.
- Flag flaky-prone areas explicitly, particularly the wait-for-preliminary-bursting and wait-for-actual-bursting steps. State the timeout/poll strategy used and why.

BUILD / COMPILE NOTES
- This repo contains a settings.xml file. Do NOT pass it to Maven when compiling or running tests locally (i.e., do not use `-s settings.xml` / `--settings settings.xml`). Using it forces Maven to re-resolve and download dependencies instead of using what's already cached in the local .m2 repository, which noticeably slows down every compile/test cycle. Compile and run with plain `mvn ...` commands using the default local settings.

WHAT NOT TO DO
- Do not reference, import, or comment-link to the analysis document or Confluence page from any test code — those are your working references only, not the repo's.
- Do not create an infra-only batch.
- Do not implement the entire matrix in one batch.
- Do not silently skip a scenario.
- Do not restructure or refactor unrelated existing tests/utilities beyond what's strictly needed.
- Do not deviate from the analysis document's reconciliation of Confluence vs. code behavior without flagging it in the batch summary.