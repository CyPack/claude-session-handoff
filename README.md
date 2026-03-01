# Claude Code Session Handoff Protocol

> Zero context loss between Claude Code sessions.

## Problem

Claude Code sessions have finite context windows. When a session ends mid-task, the next session starts with **zero knowledge** of what happened before. This leads to:

- Repeated work
- Lost architectural decisions
- Forgotten edge cases and bugs
- Broken continuity on multi-session projects

## Solution

This skill implements a **structured handoff protocol** that generates a comprehensive transition prompt at the end of each session. The next session pastes this prompt and picks up exactly where you left off.

### Key Features

- **6-step pipeline**: Collects state, updates lesson files, generates prompt
- **12-section prompt structure**: Covers context loading, scope, strategy, triggers, references, success criteria
- **50+ autonomous triggers**: Sub-agent spawning, task list creation, validation, code review, quality gates
- **3-layer context manifest**: Auto-load / mandatory read / on-demand read (token-efficient)
- **Task state serialization**: In-memory tasks survive across sessions via text snapshot
- **Lesson quality control**: Confidence tagging `[H]`/`[M]`/`[L]` with automatic cleanup
- **Manual-only activation**: No autonomous handoffs - only when you ask for it

## Why a Skill Instead of a Rule?

Claude Code has two ways to load instructions:

| | Rules (`~/.claude/rules/`) | Skills (`~/.claude/skills/`) |
|---|---|---|
| **Loading** | Auto-loaded **every** session | On-demand, loaded only when triggered |
| **Token cost** | ~8,900 tokens consumed even when unused | 0 tokens until you say "handoff yap" |
| **When useful** | Instructions needed throughout the session | Instructions needed only at specific moments |

The handoff protocol is **only needed at the end of a session** — the 2 minutes when you're wrapping up. Loading it for the entire 200K context window is wasteful:

```
200K context window = your budget for the entire session

Rule mode:    [████ 8,900 tokens wasted ████|............remaining context............]
Skill mode:   [...................full context available.................|████ loaded on demand]
```

At ~8,900 tokens, this protocol would consume **~4.5% of your context budget** sitting idle. As a skill, those tokens are available for actual work until the moment you need them.

This matters even more when you have multiple rules — we reduced our own auto-load budget from 72K to 19K tokens (74% savings) by converting infrequently-used rules to skills.

## Installation

### As a Claude Code Skill (Recommended)

```bash
# Create the skill directory
mkdir -p ~/.claude/skills/session-handoff

# Copy the skill file
cp SKILL.md ~/.claude/skills/session-handoff/SKILL.md
```

### As a Global Rule (Auto-loaded every session)

```bash
# Copy to rules directory (costs ~8,900 tokens per session)
cp SKILL.md ~/.claude/rules/session-handoff-protocol.md
```

> **Recommendation**: Use the skill approach. See [Why a Skill Instead of a Rule?](#why-a-skill-instead-of-a-rule) above.

## Usage

When you want to end a session and preserve context, say any of these:

- `"handoff yap"` / `"gecis promptu olustur"`
- `"sonraki chat icin prompt hazirla"`
- `"session bitiyor, kaydet"`
- `"devam ederiz"` / `"sonraki chat'te devam"`

Claude will:
1. Collect session state (completed tasks, open tasks, decisions, files changed)
2. Update lesson files (errors, golden paths, edge cases)
3. Generate a comprehensive transition prompt
4. Save it to `/tmp/session-handoff-{date}.md`

In the next session, paste the prompt and Claude picks up with full context.

## How It Works

### Handoff Pipeline (6 Steps)

```
STEP 1: Notify user ("Starting handoff preparation...")
STEP 2: Collect state (tasks, decisions, files, services, skills)
STEP 3: Update files (lessons → SKILL.md → SESSION-HANDOFF → MEMORY → project)
STEP 4: Integrity check (count updates, verify consistency)
STEP 5: Generate 12-section prompt with real data (no placeholders!)
STEP 6: Save to /tmp/ and display
```

### Generated Prompt Structure (12 Sections)

| Section | Purpose |
|---------|---------|
| 1. Header + Meta | Trigger reason, active skills, pipeline status |
| 2. Context Loading | 3-layer manifest (auto-load / mandatory / on-demand) |
| 3. Service Health | Health check commands for running services |
| 4. Master Scope | Session goal, in-scope/out-of-scope boundaries |
| 5. Layered Strategy | Step-by-step execution plan with success criteria |
| 5.5. Task State | Serialized in-memory tasks for recreation |
| 6. Triggers | 50+ autonomous orchestration triggers |
| 7. Parallel Tasks | Bug fixes or tasks runnable in parallel |
| 8. Security Notes | Deferred decisions, known issues |
| 9. References | File paths, endpoints, performance baselines |
| 10. Success Definition | Evidence-based completion criteria |
| 11. Open Decisions | Architectural questions needing answers |
| 12. Start Command | "GO!" with active trigger reminder |

### Autonomous Triggers (Embedded in Every Handoff)

The protocol embeds trigger tables that activate automatically in the next session:

- **Sub-agent spawning**: 3+ independent workflows? Auto-spawn agents
- **Task list creation**: 2+ tasks listed? Auto-create TaskList
- **Devil's advocate**: Architectural decision? Auto-spawn critic
- **Validation**: New endpoint? Auto-run curl test
- **Code review**: 50+ lines changed? Auto-spawn reviewer
- **Quality gates**: Phase transition? Enforce checklist

## Example Output

Here's a realistic example of what a generated handoff prompt looks like. This is what you'd paste into the next session:

<details>
<summary><strong>Click to expand full example handoff prompt</strong></summary>

```markdown
─── HANDOFF META ───
trigger: mid_task
session: 3 | protocol: v2.5
active_skills: [my-project]
pipeline: complete (4 files)
lessons: errors+1, golden+0, edge+0
coverage: feature-implementation, api-work
─── END META ───

NEW SESSION START — MyApp / User Authentication Feature
This session continues from a previous long session.
Follow the steps below IN ORDER — do not skip sections, abbreviate, or save tokens.
NOTE: This prompt is designed for a NEW (zero-context) session. If you are resuming
an existing session (claude --resume), SKIP STEPS 1-2, start from STEP 3.

═══════════════════════════════════════════════════════════════

STEP 1: SMART CONTEXT LOADING

First read the HANDOFF META block (at the top of this prompt).
- active_skills: [my-project] → load skill files in LAYER 3
- trigger: mid_task → prioritize loading the ongoing work
- pipeline: complete → all files are up to date

─── AUTO-LOADED (already in your context — do NOT Read) ───
| File | Changed This Session |
|------|---------------------|
| session-safety.md | No changes |

─── MANDATORY READ (NOT in your context) ───
1. ~/.claude/skills/my-project/lessons/errors.md
   → "Known errors" section (new: JWT refresh race condition fix)
   → [UPDATED IN HANDOFF: +1 entry]

─── ON-DEMAND READ (task type: feature-implementation) ───
1. ~/myapp/.planning/PLAN.md
   → Current phase tasks and status

═══════════════════════════════════════════════════════════════

STEP 2: SERVICE HEALTH CHECK

# Backend API
curl -s http://localhost:8000/health | python3 -m json.tool
# Expected: {"status": "ok", "db": "connected"}
# IF DOWN: cd ~/myapp && docker compose up -d

# Frontend dev server
curl -s http://localhost:3000 -o /dev/null -w "%{http_code}"
# Expected: 200
# IF DOWN: cd ~/myapp/frontend && npm run dev

═══════════════════════════════════════════════════════════════

STEP 3: THIS SESSION'S PURPOSE

Context: MyApp is a SaaS platform. Previous sessions built the database schema and
REST API skeleton. This session was implementing JWT authentication when context ran out.
The /login endpoint works, but /refresh and /logout are incomplete.

This session's SINGLE MAIN GOAL:
Complete the JWT authentication flow (refresh + logout endpoints) and add tests.

Scope boundaries:
IN SCOPE:
- Finish /api/auth/refresh endpoint (token rotation)
- Finish /api/auth/logout endpoint (token blacklisting)
- Write tests for all 3 auth endpoints
- Update API documentation

OUT OF SCOPE:
- Frontend auth UI (next session)
- OAuth/social login (milestone 2)
- Rate limiting (deferred)

═══════════════════════════════════════════════════════════════

STEP 4: LAYERED STRATEGY

Each layer executes in order. Do NOT move to Layer 2 before Layer 1 is complete.

LAYER 1 — Token Refresh (Core logic)
  Goal: Implement secure token rotation
  Task: Add /api/auth/refresh with refresh token validation,
        old token blacklisting, new token pair generation
  Success criteria: curl test returns new access + refresh tokens
  Document: Update openapi.yaml with endpoint spec

LAYER 2 — Logout (Cleanup)
  Goal: Implement token invalidation
  Task: Add /api/auth/logout with token blacklist (Redis SET)
  Success criteria: After logout, old token returns 401
  Document: Update openapi.yaml

LAYER 3 — Tests (Verification)
  Goal: Full coverage of auth flow
  Task: Write tests: happy path + expired token + reuse detection + logout
  Success criteria: pytest -v shows 8+ tests, all PASS

═══════════════════════════════════════════════════════════════

STEP 4.5: PENDING TASKS (TaskList Snapshot)

Tasks NOT completed from previous session:

| # | Subject | Status | BlockedBy | Description |
|---|---------|--------|-----------|-------------|
| 3 | Implement /refresh endpoint | in_progress | — | Token rotation with one-time use refresh tokens |
| 4 | Implement /logout endpoint | pending | 3 | Blacklist token in Redis, return 204 |
| 5 | Write auth endpoint tests | pending | 3, 4 | pytest: login, refresh, logout, edge cases |

Completed task count: 2 (DB schema, /login endpoint)

NEXT SESSION RULE: Read this table and recreate with TaskCreate.

═══════════════════════════════════════════════════════════════

STEP 5: SESSION-WIDE ACTIVE TRIGGERS

The triggers below are active THROUGHOUT this session.
When a condition is met, apply the action AUTONOMOUSLY — do NOT wait for permission.

─── MACRO SCOPE INJECTION (MANDATORY FOR ALL SUB-AGENTS) ───

Every sub-agent spawned must receive STEP 3's IN SCOPE/OUT OF SCOPE boundaries.
Spawning a sub-agent without scope injection = PROTOCOL VIOLATION.

─── SUB-AGENT SPAWN TRIGGERS ───

| Condition | Action |
|-----------|--------|
| 3+ independent workflows detected | Spawn 1 agent per workflow |
| Research + implementation + verification together | Explore + general-purpose + code-reviewer in parallel |
| 5+ files to analyze | Spawn Explore agent |
| Test + code to write simultaneously | 2 parallel agents (TDD: test agent starts FIRST) |

─── VALIDATION TRIGGERS ───

| Condition | Action |
|-----------|--------|
| New endpoint added | Write and RUN curl validation test |
| New function written | Write and RUN unit test |
| Config/environment changed | Before/after comparison test |

─── QUALITY GATE TRIGGERS ───

Plan → Impl: Plan exists + approved + test strategy exists
Impl → Test: Code written + lint passes + review done
Test → Complete: All tests PASS + evidence shown + lessons updated

═══════════════════════════════════════════════════════════════

STEP 6: PARALLEL TASKS

| Task # | File | Fix/Task | Independent? | Priority |
|--------|------|----------|-------------|----------|
| (none) | — | — | — | — |

═══════════════════════════════════════════════════════════════

STEP 7: SECURITY NOTE (NO ACTION, INFO ONLY)

- JWT_SECRET is in .env (not committed, in .gitignore)
- Redis runs without auth in dev (OK for local, NOT for production)
- Token blacklist has no TTL cleanup yet (deferred to milestone 2)

═══════════════════════════════════════════════════════════════

STEP 8: REFERENCES

Critical files:
~/myapp/src/auth/routes.py → Auth endpoints (/login done, /refresh + /logout TODO)
~/myapp/src/auth/tokens.py → JWT creation/validation helpers
~/myapp/src/auth/blacklist.py → Redis token blacklist (skeleton only)
~/myapp/tests/test_auth.py → Auth tests (2 login tests exist)
~/myapp/openapi.yaml → API spec (login documented, refresh/logout TODO)

Endpoints:
POST /api/auth/login → Returns {access_token, refresh_token} (WORKING)
POST /api/auth/refresh → Token rotation (IN PROGRESS - validation done, rotation TODO)
POST /api/auth/logout → Token blacklist (NOT STARTED)

Performance baseline:
/login response time: 45ms (local, from curl timing)

Known open bugs:
#1 JWT refresh race condition — tokens.py:34 — concurrent refresh can issue 2 valid tokens

Lesson files:
~/.claude/skills/my-project/lessons/errors.md (1 entry)
~/.claude/skills/my-project/lessons/golden-paths.md (0 entries)
~/.claude/skills/my-project/lessons/edge-cases.md (0 entries)

═══════════════════════════════════════════════════════════════

STEP 9: DEFINITION OF SUCCESS

At session end, you must have EVIDENCE-BASED answers to:

| Question | Acceptable Evidence |
|----------|-------------------|
| Does /refresh return new tokens? | curl output showing 200 + new token pair |
| Does /refresh reject reused tokens? | curl output showing 401 on second use |
| Does /logout invalidate tokens? | curl showing 204 on logout, then 401 on protected route |
| Do all tests pass? | pytest -v output showing 8+ tests, 0 failures |
| Is API spec updated? | diff showing new endpoint definitions in openapi.yaml |

"Should work" is NOT valid evidence. RUN the command, show output, THEN claim.

═══════════════════════════════════════════════════════════════

STEP 10: OPEN DECISIONS

Previous session identified this technical question:
Should refresh tokens be stored in Redis (fast, volatile) or PostgreSQL (durable, slower)?

Current approach: Redis SET with TTL matching refresh token expiry.
Trade-off: If Redis restarts, all refresh tokens invalidated (users must re-login).

Options:
- Option A: Redis only (current) — simple, fast, acceptable for this app size
- Option B: PostgreSQL + Redis cache — durable but adds complexity
- Option C: Signed refresh tokens (stateless) — no storage needed but can't revoke

Recommendation: Option A for now, revisit if user complaints arise.

───────────────────────────────────────────────────────────
START! Load context (STEP 1), check services (STEP 2),
then work toward master goal (STEP 3).
STEP 5 triggers are ACTIVE throughout — apply continuously.
Do NOT save tokens. Work thoroughly, comprehensively, autonomously.
───────────────────────────────────────────────────────────
```

</details>

> **Note**: This is a fictional example for illustration. Real handoff prompts contain actual file paths, real command outputs, and real task state from your session. The protocol never uses placeholders — every value is filled with concrete data.

## Integration with Other Systems

The protocol works alongside:

- **Skills system**: Reads/updates per-skill lesson files
- **MEMORY.md**: Updates persistent memory (architectural decisions, preferences)
- **GSD framework**: Updates `.planning/` files if active
- **MCP servers**: Tags MCP-sourced data to prevent hallucination

## Configuration

No configuration needed. The protocol is self-contained in `SKILL.md`.

Optional companion files (created automatically as you use skills):
```
~/.claude/skills/{skill}/lessons/errors.md        # Known errors + fixes
~/.claude/skills/{skill}/lessons/golden-paths.md   # Proven workflows
~/.claude/skills/{skill}/lessons/edge-cases.md     # Edge cases
~/.claude/skills/_shared/common-errors.md          # Cross-skill patterns
```

## Version History

| Version | Date | Changes |
|---------|------|---------|
| v2.5.0 | 2026-02-28 | Task state serialization, TaskList generator |
| v2.4.0 | 2026-02-28 | Manual-only triggering, per-handoff metadata |
| v2.3.0 | 2026-02-28 | 3-layer manifest context loading |
| v2.2.0 | 2026-02-28 | Credential block, lesson quality control |
| v2.1.0 | 2026-02-27 | Cross-check audit (15 fixes), 9 rules alignment |
| v2.0.0 | 2026-02-27 | Initial: 6-step pipeline, 12-section prompt, 44+ triggers |

## License

MIT

## Credits

Built iteratively by Ayaz + Claude Code (Opus) across 3 sessions, 5 iterations.
