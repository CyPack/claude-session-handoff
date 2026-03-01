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

> **Recommendation**: Use the skill approach. The protocol is only needed at session boundaries, not throughout the entire session. This saves ~8,900 tokens of context per session.

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
