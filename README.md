---
name: replatform-baseline
description: Reverse-engineer a technology-neutral baseline of user stories, business rules and existing test coverage from a Java backend repo and its UI repo, to use as acceptance criteria when re-platforming. Use before choosing the target stack. Read-only. No network, installs, MCP servers or browser.
---

# Re-platform baseline

## Goal

Describe what the current system actually does, in terms a user or another system can observe, so the same acceptance criteria can be checked against the Java app now and the new implementation later. Document the system as it is, not as it should be.

## Ground rules

1. Read-only. Never modify source code, config or tests. Write only under `docs/replatform-baseline/`.
2. No network calls, installs, MCP servers or browser use. Work from the repos and from what the user tells you.
3. Behavior, not implementation. Story text and acceptance criteria never mention classes, annotations, packages or frameworks. Code identifiers and file paths go only in Evidence fields.
4. Evidence for everything. Every story, rule and scenario cites file paths (with line ranges where useful) or test IDs. Tag anything not directly evidenced `[INFERRED]`, and anything the user told you `[CONFIRMED: user]`.
5. Record actual behavior, including bugs. Never quietly fix behavior in a story. Log suspected bugs in the quirks register.
6. Never copy secrets, credentials, connection strings, tokens or real customer data into the docs. Reference the file and describe the shape instead (for example "10-digit account number").
7. Don't guess. If you can't tell, raise an open question.
8. Improvement ideas go in `ideas.md`, never in stories.
9. Stay inside the agreed area. Note anything out of scope in `progress.md` for later.

## How to work

Work in phases. At each CHECKPOINT, summarize what you found in under 15 lines, say what you'll do next, and wait for the user.

For large codebases, map entry points across everything at summary level first, then go deep one area at a time. An area is a business capability agreed with the user (for example "trade capture" or "user admin"), with a short code like TRD or ADM. If your environment supports parallel subagents, use one per module and merge results into the same files with consistent IDs.

## Phase 0: Scope

Ask one question at a time, at most 7, and give your recommended answer with each:

1. Paths to the backend and UI repos, and which area to start with. Recommend one high-value area.
2. Who uses the system (roles, teams) and the 3 to 5 most critical journeys.
3. Known batch jobs, reports, integrations and manual workarounds around the app.
4. Known bugs or odd behaviors that users rely on.
5. Where else test cases live (for example Jira test management, Confluence, spreadsheets). If the user wants them included, ask them to export into `docs/replatform-baseline/inputs/`.
6. One example of a good user story from their team, to match its style. If none, use the template in Phase 3.
7. Whether you may run the existing test build (for example `mvn test` or `gradle test`) to record pass/fail. Default: no.

Then create `README.md` (scope, repo paths, the current commit hash of each repo from `git rev-parse HEAD` or from the user, file index) and `progress.md`.

## Phase 1: System map

Find every way behavior gets triggered, not just what the UI calls.

`system-map/ui-inventory.md` (IDs UI-001...)

| ID | Screen / route | Roles that see it | User actions | Client-side validation | Backend calls | Source |

Find routes from router configs, page components or server-side templates. Include UI that changes by role or feature flag.

`system-map/entry-points.md` (IDs EP-001...)

| ID | Entry point | Type | Trigger | Used by UI (UI IDs) | Observable effects | Source |

Types: HTTP endpoints (REST, SOAP, MVC controllers), scheduled jobs (cron, Quartz, scheduler annotations), batch jobs, message listeners (JMS, Kafka, RabbitMQ), file imports and exports, outbound calls (HTTP clients, SFTP, email), startup tasks, database triggers and stored procedures, migration scripts that seed reference data. Observable effects are responses, data written (tables), messages sent, files produced, emails sent and calls to other systems. Mark entry points the UI never reaches as DARK. They still need stories.

`system-map/roles-and-access.md`: roles, how users get them, and which screens and entry points each role can use. Look at security config, role checks and permission tables.

`system-map/cross-cutting.md`: login and session timeout, global error handling (which failures become which HTTP status and message), validation rules and messages, audit logging, pagination and sort defaults, caching that users can notice, upload limits, time zone handling, translations, and persistence side effects (entity listeners, cascades, soft deletes, audit columns, optimistic locking).

CHECKPOINT.

## Phase 2: Test inventory

Find every existing test in both repos: unit and integration tests (JUnit, TestNG, Spock, `*Test`, `*IT`, Spring and Testcontainers tests), API tests (Postman collections, REST Assured), end-to-end tests (Selenium, Cypress, Playwright, Protractor), BDD feature files, UI unit tests (Jest, Karma), performance tests, and anything in `inputs/`.

`tests/test-inventory.md` (IDs T-001...; also write `tests/test-inventory.csv` if there are more than 100 tests)

| ID | Location (file#method) | Framework | Level (unit / integration / API / E2E / manual) | Behavior verified, in plain language | Key assertions and concrete values | Portability | Status | Story IDs |

Portability:
- PORT: drives the system from outside (UI, HTTP, messages, files, shared database) and checks observable results. Can be rewritten as an acceptance scenario.
- MINE: tests internal classes. Extract its business rules and example values, but don't carry the test forward.
- DROP: tests framework wiring, getters and setters, or only mocks. Count these; don't list each one.

Status: enabled, disabled (for example `@Disabled`, `@Ignore`, or excluded in build config), pass, fail or not run. List disabled tests separately, since they often mark known bugs. Treat rules that come only from heavily mocked tests as lower confidence, because the mock may define behavior that production doesn't have.

`business-rules.md` (IDs BR-<AREA>-001...)

| ID | Rule in plain language | Concrete examples | Evidence (code, test IDs) | Verified by a test? |

Pull rules from assertions, validators, calculations, state transitions and conditionals.

CHECKPOINT.

## Phase 3: User stories

One area at a time, written to `stories/<area>.md`. Group by user goal, not by screen or class.

    ### US-<AREA>-<NNN>: <short title>
    As a <role>, I want <capability>, so that <benefit>.
    Risk: High | Medium | Low
    Confidence: High | Medium | Low

    Acceptance criteria
      @from-code @from-test:T-012
      Scenario: <one observable behavior>
        Given <starting state, with concrete data>
        When <action or trigger>
        Then <observable outcome, with concrete values>

      Scenario Outline: <rule with several cases>
        Given ...
        When ...
        Then ...
        Examples:
          | input | expected |

    Business rules: BR-...
    Quirks: QK-...
    Evidence: UI-..., EP-..., <file paths and line ranges>
    Existing tests: T-...   Coverage: COVERED | PARTIAL | RULES-ONLY | UNCOVERED
    Open questions: OQ-...

Rules for writing them:
- Tag the "so that" part `[INFERRED]` until the user confirms it.
- Risk is High for anything touching money, regulatory reporting, data integrity or security.
- Outcomes must be observable: what a screen shows, what an API returns (status and fields), files produced (name pattern, columns), messages or emails sent, and persisted data when other systems read it or the database is being kept.
- Use concrete values from code, tests and fixtures: limits, formats, rounding, error messages, defaults, sort orders.
- Cover the unhappy paths: validation errors, permission denied, not found, conflicts, timeouts.
- For scheduled, batch and integration behavior, write from the view of the person or downstream system that depends on it, and put the trigger in the Given (for example "Given the end-of-day job runs at 02:00 on business days").
- Put role differences in separate scenarios. Make separate stories only when the goal differs.
- Coverage: COVERED means a PORT test exercises the main scenario, PARTIAL means some scenarios, RULES-ONLY means only MINE tests, UNCOVERED means no tests.

Parity hot spots. Java and Python behave differently here, so wherever the app touches one, pin it with explicit examples:
- Decimal math: scale, rounding mode and where rounding happens. Python's `round()` and `decimal` default to half-even; Java code often uses HALF_UP.
- Integers: Java int and long overflow wraps, Python's doesn't. Division and modulo with negative numbers give different results.
- Number formatting: Java prints 1.0E10 where Python prints 10000000000.0.
- Dates and times: time zones, daylight saving, business-day calendars, how lenient date parsing is, default formats. Java timestamps can carry nanoseconds; Python's datetime stops at microseconds.
- Text: sorting and collation, case-insensitive matching, trimming, Unicode, default character encoding, regex behavior.
- Empty values: null vs empty string vs a missing JSON field.
- JSON: field names, whether null fields appear, number and date formats, field order if any consumer depends on it.
- Ordering: any list a user sees, especially where code relies on hash map iteration order.
- Errors: HTTP status codes, error body shape and message text.
- Transactions: what's all-or-nothing, and what happens on concurrent edits.

`quirks.md` (IDs QK-001...)

| ID | Observed behavior | Evidence | Looks like a bug? | Decision: KEEP / FIX / ASK |

Default decision is ASK.

CHECKPOINT after each area.

## Phase 4: Gap interview

Only ask what the code and tests can't answer, so check them first. One question at a time, highest risk first. For each question give:

1. Why it matters, in one line.
2. What the code shows, with file references.
3. Your recommended answer.

Then wait. Cover the "so that" for High-risk stories, rules with no test, quirk decisions, possibly dead features (is anyone still using this?), DARK entry points (who depends on them?) and what integrations are expected to do. Record answers in the stories, tagged `[CONFIRMED: user]`, and in `decisions.md`. If the user says "batch", list up to 10 questions at once. Put anything unanswered in `open-questions.md`:

| ID | Question | Why it matters | Recommended answer | Owner | Status |

## Phase 5: Traceability and coverage

`traceability-matrix.csv` with columns: Story ID, Rule IDs, UI IDs, Entry point IDs, Test IDs, Coverage, Risk, Confidence.

`coverage-summary.md`: counts by coverage and risk; High-risk stories that are UNCOVERED or RULES-ONLY (these need new tests against the Java app before any rewrite); DARK entry points; dead-feature candidates; disabled tests; top open questions.

If the user wants the stories in Jira, also write `jira-import.csv` (Summary, Description, Labels) for Jira's CSV import.

Self-check before calling an area done. Fix what fails, then report:
- Every UI screen, entry point and PORT test maps to at least one story, or is listed as excluded with a reason.
- No code identifiers in story text or scenarios. Search for class-name suffixes (Service, Controller, Repository, Dao, Impl), `@` annotations and package names outside Evidence fields.
- Every scenario has concrete values and an observable outcome.
- Everything not evidenced or confirmed is tagged `[INFERRED]`.
- `progress.md` states what's done and the exact next step, so a new session can pick up.

## Output layout

    docs/replatform-baseline/
      README.md              scope, repo paths, commit hashes, index
      progress.md            phase status and next step
      inputs/                exports from Jira, Confluence, spreadsheets
      system-map/            ui-inventory.md, entry-points.md, roles-and-access.md, cross-cutting.md
      tests/                 test-inventory.md (and .csv)
      business-rules.md
      stories/<area>.md
      quirks.md
      open-questions.md
      decisions.md
      ideas.md
      traceability-matrix.csv
      coverage-summary.md

## Finish

Report counts (stories, rules, tests by portability, stories by coverage), the top 5 risks and the top 5 open questions. Then state the next step: turn the acceptance criteria into executable tests and run them against the Java app. The baseline only counts once they pass there.
