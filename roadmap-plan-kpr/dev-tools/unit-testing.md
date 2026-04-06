# Unit Testing — Baseline Definition

> Owner: KPR
> Scope: Unit testing infrastructure + baseline coverage for trybot-api (Python) and trybot-ui (React Native)
> Current completion: ~5% — 11 test files in backend (all integration, no unit tests), 1 test file in frontend (renders App), zero CI integration, zero coverage tracking
> Context: Part of "Agentic Coding as a Team" dev-tools initiative. Other dev-tools owned by AKN (foundation setup, planning, system design) and RKP (development, review).

---

## What This Is

Unit testing is the automated proof that individual pieces of the system work correctly in isolation. It answers: "if I change this function, will something break?" — without running the entire system, calling external APIs, or connecting to a database.

Right now, both trybot-api and trybot-ui deploy directly to production without running any tests. A developer can merge broken code, and the only safety net is manual testing or users discovering bugs. Unit testing creates the first automated quality gate: code that breaks existing behavior cannot be merged.

**Why this matters for an agentic coding team:** The dev-tools initiative automates the ticket-to-PR pipeline using Claude Code. Automated agents generating code without automated tests is a recipe for shipping bugs at machine speed. Unit tests are the guardrail that makes AI-assisted development safe.

## Current State

### Backend (trybot-api — Python/FastAPI)

| What Exists | What's Missing |
|---|---|
| 11 test files in `tests/` directory | Actual unit tests (all existing tests are integration) |
| `conftest.py` with Supabase session fixtures | Isolated unit tests with mocks (no DB required) |
| Pytest available in venv (not in requirements.txt) | Pytest in dependencies + `pytest.ini` config |
| Manual test scripts (`tests/manual_scripts/`) | `pytest-asyncio` for async handler tests |
| Mock telephony provider template (unused) | `pytest-mock` / `unittest.mock` usage |
| Firebase/DB test route endpoints | Test data factories/builders |
| | `pytest-cov` for coverage reporting |
| | Any test execution in CI/CD pipeline |
| | Coverage thresholds or quality gates |

### Frontend (trybot-ui — React Native)

| What Exists | What's Missing |
|---|---|
| Jest 29.7.0 configured with `react-native` preset | More than 1 test file |
| `react-test-renderer` installed | `@testing-library/react-native` |
| `npm test` script defined | Component-level tests |
| `jest.config.js` exists (minimal) | Mock providers (Redux, Navigation, Firebase) |
| 1 test: `App.test.tsx` (renders App) | `setupTests.ts` for global test setup |
| | Coverage configuration or thresholds |
| | Any test execution in CI/CD pipeline |

### CI/CD Integration

| Pipeline | Current Behavior |
|---|---|
| `trybot-api/.github/workflows/deploy.yml` | Build + deploy directly. No test step. |
| `trybot-ui/.github/workflows/android-build.yml` | Build APK directly. No test step. |
| `trybot-ui/.github/workflows/ios-build.yml` | Build IPA directly. No test step. |

## Baseline Definition

**The line is:** Both projects have test infrastructure configured, critical service functions are covered by isolated unit tests (no external dependencies), tests run automatically in CI before every deploy, and coverage is tracked with a minimum threshold. Anything beyond this is refinement.

---

## Baseline Components

### 1. Backend Test Infrastructure — ~3 dev days

The backend has no unit testing setup. All existing tests connect to real Supabase. The first step is creating infrastructure that enables isolated, fast, repeatable tests.

#### 1a. Pytest configuration and dependencies

Add to `requirements.txt` (or `pyproject.toml`):
```
pytest>=8.0
pytest-asyncio>=0.23
pytest-cov>=4.1
pytest-mock>=3.12
httpx>=0.27  # for TestClient async support
```

Create `pytest.ini` (or `pyproject.toml [tool.pytest]`):
- Test discovery: `tests/unit/`, `tests/integration/`
- Default: run only unit tests (`-m "not integration"`)
- Async mode: `auto` for `pytest-asyncio`
- Coverage: `--cov=app --cov-report=term-missing --cov-fail-under=40`

**Why baseline:** Without config, every developer invents their own test commands. Config standardizes: `pytest` runs unit tests, `pytest -m integration` runs integration tests.

#### 1b. Test directory structure

```
tests/
├── unit/                      # Fast, isolated, no external deps
│   ├── conftest.py            # Shared fixtures (mock DB, mock Redis, mock HTTP)
│   ├── services/              # Service function tests
│   ├── routes/                # Route handler tests (TestClient)
│   └── utils/                 # Utility function tests
├── integration/               # Slow, needs real services (existing tests move here)
│   ├── conftest.py            # Supabase fixtures (existing)
│   └── ...                    # Existing test files move here
└── fixtures/                  # Shared test data (JSON payloads, mock responses)
```

**Why baseline:** Separating unit from integration means `pytest` in CI runs in seconds (unit only), not minutes (waiting for Supabase). Developers run unit tests locally on every change; integration runs on merge.

#### 1c. Core mock fixtures

Create reusable mock fixtures in `tests/unit/conftest.py`:

- **Mock Supabase client** — returns canned responses for `safe_select`, `safe_insert`, `safe_update`, `safe_delete`. No real DB connection. Why: every service function calls the database. Without a mock DB, you can't test any service in isolation.

- **Mock Redis client** — in-memory dict that mimics `get`/`set`/`hset`/`hget`/`setnx`. Why: session manager, rate limiting, caching all use Redis. `fakeredis` library is an option.

- **Mock HTTP client** — returns canned responses for Twilio, Meta, OpenAI, etc. Why: every outbound call (LLM, telephony, WhatsApp) must be mockable. Use `pytest-mock` to patch `httpx.AsyncClient`.

- **Mock LLM gateway** — returns deterministic responses for `generate()`, `stream()`. Why: conversation engine, planning, and 15+ services call the LLM. Tests must not hit real APIs.

- **Auth fixture** — provides a mock `AuthenticatedUser` object with `user_id`, `profile_id`, RLS-scoped mock client. Why: every route handler requires auth context.

**Why baseline:** Mock fixtures are the foundation. Without them, every unit test devolves into an integration test. Building fixtures once means every future test gets them for free.

---

### 2. Backend Unit Tests for Critical Services — ~5 dev days

Not every function needs a test at baseline. Target the functions where bugs cause the most damage: side effects (calls, messages), money (billing, usage), and data integrity (contacts, sessions).

#### Priority 1 — Side-effect services (bugs = duplicate calls/messages)

| Service | Functions to Test | Why Critical |
|---|---|---|
| `call_service.py` | `create_outbound_call()`, `handle_status_webhook()` | Wrong status = call stuck forever |
| `whatsapp_service.py` | `send_message()`, `process_webhook()`, `generate_auto_reply_text()` | Lost message = silent failure |
| `telephony_factory.py` | Provider selection, failover logic | Wrong provider = call fails |
| `conversation_service.py` | `generate_call_summary()`, `generate_ai_agent_intro()` | Bad summary = lost context |

#### Priority 2 — Data integrity services (bugs = data corruption)

| Service | Functions to Test | Why Critical |
|---|---|---|
| `database_utils.py` | `safe_insert()`, `safe_select()`, `safe_update()`, `safe_delete()`, `bulk_insert()` | Every component uses these |
| `contacts_service.py` | `resolve_contact_from_prompt()`, contact CRUD | Wrong resolution = wrong person called |
| `user_service.py` | Profile CRUD, OAuth token lifecycle | Broken auth = locked out |
| `session_manager.py` | Create, read, write, expire | Lost session = call drops context |

#### Priority 3 — Configuration and routing (bugs = wrong behavior)

| Service | Functions to Test | Why Critical |
|---|---|---|
| `settings.py` | Config loading, validation, provider selection | Wrong config = everything breaks |
| `intent_service.py` | Intent classification | Wrong intent = wrong action during call |
| `search_handlers.py` | Provider routing, fallback chain | Failed search = agent has no info |

**Test pattern for each:**
1. Test the happy path (correct input → correct output)
2. Test edge cases (empty input, missing fields, null values)
3. Test error handling (provider returns 500, DB returns error)
4. Test idempotency where relevant (duplicate webhook → same result)

**Why baseline:** These ~15 service files contain the core business logic. Covering them catches 80% of bugs with 20% of the effort. The rest of the codebase is plumbing that rarely breaks.

---

### 3. Frontend Test Infrastructure — ~2 dev days

The frontend has Jest installed but nothing configured. One test exists. The priority is setting up infrastructure that makes writing tests easy, then covering the most critical components.

#### 3a. Install testing dependencies

```
npm install --save-dev @testing-library/react-native @testing-library/jest-native jest-expo
```

Why `@testing-library/react-native` over `react-test-renderer`: testing-library tests user behavior ("tap the button labeled 'Call'") not implementation details ("check the 3rd child of the 2nd View has prop X"). Tests survive refactors.

#### 3b. Configure Jest properly

Update `jest.config.js`:
- Module name mapper for `@/` path aliases
- Transform ignore patterns for native modules
- Setup file: `setupTests.ts`
- Coverage collection from `src/` (or equivalent)
- Coverage threshold: `{ global: { lines: 30 } }`

#### 3c. Create `setupTests.ts` with mock providers

Global mocks that every test needs:

- **Mock Redux store** — `configureStore()` with preloaded state. Why: every connected component needs a store.
- **Mock Navigation** — `jest.mock('@react-navigation/native')` with `useNavigation`, `useRoute`. Why: most screens use navigation.
- **Mock Firebase** — `jest.mock('@react-native-firebase/messaging')`, `jest.mock('@react-native-firebase/analytics')`. Why: Firebase initializes on import; must be mocked.
- **Mock AsyncStorage** — `jest.mock('@react-native-async-storage/async-storage')`. Why: Redux Persist uses it.
- **Mock Supabase client** — `jest.mock('../services/supabase')`. Why: auth and API calls.
- **Render helper** — `renderWithProviders(component, { preloadedState, route })` that wraps component in Redux Provider + NavigationContainer + ThemeProvider.

**Why baseline:** Without these mocks, attempting to render any screen crashes immediately on Firebase init, Redux Provider missing, or navigation context missing. The setup file removes this barrier.

---

### 4. Frontend Unit Tests for Critical Components — ~3 dev days

#### Priority 1 — Core interaction flows

| Component | What to Test | Why Critical |
|---|---|---|
| `AIChatScreen.tsx` | Renders message list, sends message on submit, shows loading state | Primary interaction surface |
| `agentRunReducer.ts` | Reduces SSE events to correct UI state (plan, steps, approvals) | Logic-heavy; drives all run visualization |
| `sseFetchClient.ts` | Parses SSE frames, handles reconnection | If this breaks, entire chat is dead |
| `ContactTaggingInput` | @mention resolution, suggestion display | If broken, can't tag contacts in messages |

#### Priority 2 — Redux slices (pure logic, easy to test)

| Slice | What to Test | Why Critical |
|---|---|---|
| `chatListSlice.ts` | Add/remove/update sessions, stale-while-revalidate | Chat session integrity |
| `contactsSlice.ts` | Sync, search, dedup | Contact data integrity |
| `taskHistorySlice.ts` | Tab filtering, status updates | Task tracking accuracy |
| `profileSlice.ts` | Profile load, update | User identity |
| `dndSlice.ts` | Toggle, schedule, snooze logic | DND correctness |

#### Priority 3 — Service functions (pure logic)

| Service | What to Test | Why Critical |
|---|---|---|
| `chatService.ts` | Message formatting, upload handling | Data sent to backend |
| `contactService.ts` | Device sync diff, dedup | Contact merge correctness |
| `voiceTaskCreationService.ts` | Task payload construction | Task creation correctness |

**Why baseline:** Redux slices are pure functions — easiest to test, highest reliability gain. The `agentRunReducer` is the most complex piece of frontend logic and drives the entire chat UI.

---

### 5. CI/CD Integration — ~2 dev days

Tests that don't run automatically are tests that get ignored. The single most important infrastructure change: add a test step before deploy in all three CI pipelines.

#### Backend CI (`deploy.yml`)

Add before the deploy step:
```yaml
- name: Run unit tests
  run: |
    pip install -r requirements.txt
    pip install pytest pytest-asyncio pytest-cov pytest-mock
    pytest tests/unit/ --cov=app --cov-report=term-missing --cov-fail-under=40
```

If tests fail → pipeline fails → deploy blocked. No exceptions.

#### Frontend CI (`android-build.yml`, `ios-build.yml`)

Add before the build step:
```yaml
- name: Run unit tests
  run: |
    npm ci
    npm test -- --coverage --coverageThreshold='{"global":{"lines":30}}'
```

Same rule: tests fail → build blocked.

#### Coverage thresholds

| Project | Initial Threshold | Target after 3 months |
|---|---|---|
| trybot-api | 40% line coverage on tested files | 60% overall |
| trybot-ui | 30% line coverage on tested files | 50% overall |

Start low and ratchet up. A threshold that fails on existing code gets disabled day one.

**Why baseline:** CI integration is the difference between "we have tests" and "tests protect us." Without CI, tests are a developer's personal choice. With CI, they're a team guarantee.

---

### 6. Test Writing Guide — ~1 dev day

A short guide (in `CLAUDE.md` or `tests/README.md`) so any developer (or Claude Code agent) can write tests consistently.

Contents:
- **Where tests live:** `tests/unit/` for fast isolated tests, `tests/integration/` for real-service tests
- **How to run:** `pytest` (unit), `pytest -m integration` (integration), `npm test` (frontend)
- **Naming:** `test_{function_name}_{scenario}_{expected}` — e.g., `test_send_message_expired_window_requires_template`
- **What to mock:** External services (DB, Redis, HTTP, LLM) — always. Internal functions — rarely.
- **What NOT to test:** Private methods, framework internals, third-party library behavior
- **Test pattern:** Arrange (setup) → Act (call function) → Assert (check result)
- **Fixtures available:** List of mock fixtures with usage examples

**Why baseline:** Without a guide, test style diverges across developers. The Claude Code `/test-write` skill needs clear patterns to generate consistent tests.

---

## What Counts as Refinement (explicitly deferred)

| Refinement | Why Deferred |
|---|---|
| End-to-end (E2E) tests | Require full system running. Unit tests are the foundation — E2E layers on top. |
| Snapshot testing (frontend) | Brittle — breaks on any UI change. Better as a conscious choice per component. |
| Visual regression testing | Requires screenshot comparison tooling (Chromatic, Percy). High setup cost. |
| API contract testing (OpenAPI diff) | Covered by AKN's foundation-setup-claude.md dev-tools. Not unit testing scope. |
| Mutation testing | Measures test quality (not code quality). Valuable but requires mature test suite first. |
| Performance/load testing | Different category — tests system behavior under load, not correctness. Scripts exist in `scripts/`. |
| Test data factories (faker, factory_boy) | Manual test data is acceptable at baseline. Factories pay off when test count is high. |
| Property-based testing (Hypothesis) | Advanced technique for edge-case discovery. Unit tests cover known cases first. |
| Database integration test isolation (test containers) | Existing Supabase fixtures work for integration tests. Containerized isolation is scale. |
| Frontend E2E (Detox, Maestro) | Requires device/simulator. Unit + component tests come first. |
| Cross-repo test orchestration | Both repos test independently at baseline. Orchestration is for when contracts diverge. |

## Dependencies

| Needs | From | Why |
|---|---|---|
| Backend codebase access | trybot-api repo | Write tests against actual service functions |
| Frontend codebase access | trybot-ui repo | Write tests against actual components |
| CI/CD pipeline access | GitHub Actions | Add test steps to workflows |
| Mock data for fixtures | Service API contracts | Need to know what DB/API responses look like |
| CLAUDE.md test section | AKN (foundation-setup) | Claude Code `/test-write` skill reads this for patterns |

## Effort Estimate

| Component | Effort |
|---|---|
| Backend test infrastructure (config, structure, fixtures) | 3 dev days |
| Backend unit tests (~15 critical services) | 5 dev days |
| Frontend test infrastructure (config, setup, mocks) | 2 dev days |
| Frontend unit tests (reducers, slices, components) | 3 dev days |
| CI/CD integration (3 pipelines + coverage gates) | 2 dev days |
| Test writing guide | 1 dev day |
| **Total** | **~16 dev days** |

## Build Order

```
Phase 1 — Infrastructure (nothing can be tested without this)
├── Backend: pytest config + directory structure + mock fixtures     3 days
└── Frontend: Jest config + setupTests + mock providers             2 days

Phase 2 — Critical Tests (highest-risk code first)
├── Backend: Side-effect services (calls, WhatsApp, telephony)      2.5 days
├── Backend: Data integrity services (DB utils, contacts, sessions) 2.5 days
└── Frontend: Redux slices + agentRunReducer + SSE client           3 days

Phase 3 — CI Gate (make tests mandatory)
├── Add test step to deploy.yml (backend)                           0.5 days
├── Add test step to android-build.yml + ios-build.yml (frontend)   0.5 days
├── Coverage thresholds (40% backend, 30% frontend)                 0.5 days
└── Test writing guide                                              1 day (overlap)
```

## Overlap with Other Dev-Tools

| Area | Unit Testing (KPR) Owns | Other Tool Owns | Interface |
|---|---|---|---|
| Test generation | Test patterns, fixtures, coverage thresholds | `/test-write` Claude skill (AKN foundation) | Unit testing defines patterns → `/test-write` generates tests following those patterns |
| Code review | Test coverage checks in CI | `/self-review` + Code Review SubAgent (RKP) | CI reports coverage; review agent checks test presence on changed files |
| API contracts | Unit tests mock API responses | API Contract Testing (AKN foundation) | Contract tests validate schema; unit tests validate behavior |
| Test execution | Unit + integration in CI | `/test-run` Claude skill (AKN foundation) | Unit testing configures pytest/jest; `/test-run` invokes them |
| Quality gate | Coverage threshold in pipeline | `/quality` Claude skill (AKN foundation) | Both contribute to the PR quality gate; coverage is one of several checks |

## Key Concepts Explained

### What is the difference between unit and integration tests?
A **unit test** tests a single function in isolation — all external dependencies (database, Redis, APIs) are replaced with mock objects that return canned responses. It runs in milliseconds, needs no infrastructure, and tells you exactly which function broke. An **integration test** tests multiple components working together against real services — it's slower, needs infrastructure (Supabase, Redis), and tells you the system works end-to-end. Both are needed; unit tests are the foundation.

### Why mock external services?
If your test calls real Supabase, it fails when Supabase is down, when the network is slow, when test data changes, or when another test modifies the same rows. Mocking means: "when the code calls `safe_select('contacts', ...)`, return this canned list." The test is fast (no network), reliable (no external state), and isolated (no other test can interfere). You test your logic, not Supabase's availability.

### What is coverage and why 40%?
Coverage measures "what percentage of your code runs during tests." 40% on a codebase with 0% is ambitious but achievable — it forces you to cover the critical paths. 100% is a vanity metric — testing trivial getters wastes time. The goal is meaningful coverage of high-risk code, not a number. Start at 40%, ratchet to 60% as the test suite matures.

### Why separate unit and integration test directories?
Speed. Unit tests run in 2-5 seconds. Integration tests run in 30-120 seconds (waiting for DB, APIs). In CI, you want the fast gate first: if unit tests fail, don't waste time on integration. Locally, developers run unit tests on every save; integration before committing. The directory split makes this `pytest tests/unit/` vs `pytest tests/integration/` — simple and clear.

### How does this enable AI-assisted development?
The dev-tools initiative uses Claude Code to automate ticket-to-PR. When Claude writes code, it also needs to write tests (via `/test-write`). Without test infrastructure, there's nothing to generate. With pytest config, fixtures, and patterns in place, Claude can: (1) import the mock fixtures, (2) follow the naming convention, (3) test the function it just wrote, (4) run `pytest` to verify. The test writing guide in CLAUDE.md is what Claude reads to know how to generate tests for this specific codebase.
