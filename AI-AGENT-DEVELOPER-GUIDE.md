# AI Agent Developer Guide

How to work effectively with AI coding agents in large codebases.

This guide is for developers who use AI agents (Cursor, Copilot, Aider, etc.) as part of their daily workflow. Six topics — each one is a principle, the problem it solves, and the solution with examples.

---

## 1. Write everything down — agents have no memory

### The principle

An AI agent has no memory of its own. The folder structure, the docs, the status files — that IS the agent's knowledge. If something isn't written down in the repo, it doesn't exist for the agent. Every session starts from zero unless you give it something to read.

```
What you know          What the agent knows
┌──────────────┐       ┌──────────────┐
│ Architecture │       │              │
│ Conventions  │       │  Only what's │
│ Past bugs    │──X──> │  in files    │
│ Decisions    │       │              │
│ Context      │       │              │
└──────────────┘       └──────────────┘
     Your head              The repo

X = Everything NOT written down is lost to the agent
```

### The problem: Session amnesia

Every new chat session starts from zero. The agent doesn't remember what it fixed yesterday, what approach was tried and failed, or what the current priority is. You re-explain the same context every time.

```
Session 1                Session 2                Session 3
    │                        │                        │
    ├─ Found bug X           ├─ Found bug X (again)   ├─ Found bug X (again!)
    ├─ Analyzed root cause   ├─ Analyzed root cause   ├─ Analyzed root cause
    ├─ Fixed it              ├─ Fixed it differently  ├─ "Let me try..."
    └─ Chat closed           └─ Chat closed           └─ Same work, 3rd time
```

### The solution: Persistent agent workspace (`status/`)

Give the agent a memory that outlives any single chat. A `status/` folder acts as shared state between sessions. Progress lives in files, not in chat history.

```
status/
├── BUGS.md     # Every known bug: ID, root cause, affected files, severity
├── DEV.md      # Fix plan: grouped tasks, implementation steps, checkboxes
└── README.md   # Instructions for the agent on how to use this folder
```

```
Without status/                      With status/

Session 1: Fix bugs A, B, C         Session 1: Fix bugs A, B, C
Session 2: "What bugs?" ──> explain  Session 2: Reads DEV.md
Session 3: "What bugs?" ──> explain      │
Session 4: "What bugs?" ──> explain      ├─ [x] Bug A ── done
                                         ├─ [x] Bug B ── done
  You re-explain every time.             ├─ [ ] Bug C ── start here
                                         └─ [ ] Bug D ── next
                                    
                                     Agent starts on Bug C immediately.
```

**Bad:** Each session, you type: "There's a bug in module A where the retry logic fails on 429 responses. I already tried approach X but it didn't work because..."

**Good:** `status/BUGS.md` has:

```markdown
## BUG-003: Retry logic fails on 429 responses
- **File:** src/libs/http_client.py
- **Root cause:** Retry count resets on redirect before 429
- **Tried:** Approach X — failed because Y
- **Severity:** High
```

And `status/DEV.md` has:

```markdown
## FIX-02: Fix retry logic
**Bugs:** BUG-003
**Steps:**
- [x] Analyze retry flow
- [ ] Move retry counter above redirect handler
- [ ] Add test for 429 → retry → success
```

New session reads this and starts coding in 10 seconds.

---

## 2. Document every strategy — long chats decay

### The principle

Every time you solve a non-trivial problem — a new architecture, a refactor approach, a workaround — write down **what** you did and **why** you did it. Code shows what was done. Only docs explain why. If the agent doesn't know why you chose approach A over approach B, it will confidently refactor your code back to approach B.

```
Session 1                          Session 2

You + Agent                        New Agent
    │                                  │
    ├─ Work out strategy ──┐           ├─ Reads docs
    ├─ Implement it        │           │    │
    ├─ Code works          │           │    ├─ Sees strategy
    │                      │           │    ├─ Sees reasoning
    └─ Close chat ─────────┤           │    └─ Continues forward
                           │           │
            ┌──────────────┘           │
            ▼                          │
     ┌─────────────┐                   │
     │ Update docs │──────────────────>│
     │ with WHY    │
     └─────────────┘

Without the docs update, Session 2 re-derives or undoes Session 1's work.
```

### The problem: Context decay in long sessions

As a chat session gets longer, the agent's responses degrade. It becomes less accurate, contradicts itself, and forgets earlier decisions. The context window is finite — early messages get pushed out as new ones pile in.

```
Chat message 1-20:     High quality. Agent remembers everything.
Chat message 20-50:    Good. Slight repetition.
Chat message 50-100:   Declining. Starts contradicting earlier work.
Chat message 100+:     Poor. Forgets key decisions. Generic output.

     Quality
       │
  High ├──────╲
       │       ╲
       │        ╲──────╲
       │                ╲──────────────
  Low  │
       └───────────────────────────────
       0       50       100      Messages
```

### The solution: Document-then-rotate workflow

When a chat decays, don't fight it — save context to files and start fresh. But this only works if the important context isn't trapped inside the dying chat.

```
Long chat decaying?
        │
        ▼
┌─ Signs of decay: ──────────────────────────────────┐
│  • Agent repeats itself or contradicts earlier work │
│  • Suggestions become generic or off-target         │
│  • Agent "forgets" decisions from earlier in chat   │
│  • Fixes break things the agent already fixed       │
└─────────────────────────────────────────────────────┘
        │
        ▼
┌─ Before closing: ─────────────────────┐
│  1. Update DEV.md checkboxes          │
│  2. Write strategy/reasoning to docs  │
│  3. Capture any tricky decisions      │
└───────────────────────────────────────┘
        │
        ▼
┌─ Start fresh chat: ──────────────────┐
│  "Read status/DEV.md, continue."     │
│                                      │
│  Agent picks up where you left off   │
│  with a clean context window.        │
└──────────────────────────────────────┘
```

**Bad:** You spend 40 minutes migrating from library X to library Y. Code is done. You close the chat. Next session sees library Y imports but doesn't know why — suggests "simplifying" back to library X.

**Good:** Before closing, you update the docs: "HTTP client is library Y with stealth mode. We migrated from library X because targets detect X's default fingerprint and block it." Next session builds on the decision instead of undoing it.

**The habit:** Every big implementation or strategy change → update docs → safe to start fresh anytime. The more you document, the cheaper fresh chats become.

---

## 3. Organize for navigation — agents can't see everything

### The principle

An AI agent cannot "see" your entire project at once. A codebase with 50+ files is too large for any context window. The goal isn't to restrict the agent — it's to organize the codebase so it doesn't waste effort searching. Name things for discovery: number your docs, name files literally, use prefixes like `base_` for parent classes.

```
Your project: 80 files

  ┌─────────────────────────────────────────────────┐
  │ file1  file2  file3  file4  file5  file6  file7 │
  │ file8  file9  file10 file11 file12 file13 file14│
  │ file15 file16 file17 file18 file19 file20 ...   │
  └─────────────────────────────────────────────────┘
                        │
          Agent's context window: 5-10 files
                        │
                ┌───────┴───────┐
                │ file3  file7  │  ← Agent sees these
                │ file12 file15 │
                └───────────────┘

  The other 76 files? The agent guesses.
```

### The problem: Flat structure confusion

When your code is organized as a sea of files at the same level, the agent has no way to prioritize what to read first. It spends most of its token budget exploring, not fixing.

```
project/
├── a.py
├── b.py
├── c.py
├── d.py            Agent: "Which of these 50 files
├── e.py                    is the one I need?"
├── ...                     *reads 20 of them*
├── x.py                    "Still looking..."
├── y.py                    *reads 15 more*
└── z.py                    "Found it. But I used most
                             of my context window exploring."
```

### The solution: Organized hierarchy + numbered `docs/`

Structure tells the agent where to look. Numbered docs tell it what's where. Descriptive names tell it what everything does.

```
Bad: Flat structure                Good: Organized structure

project/                           project/
├── utils.py                       ├── src/
├── helpers.py                     │   ├── libs/        # Shared — read first
├── common.py                      │   ├── modules/     # One folder per module
├── module_a.py                    │   │   ├── a/
├── module_b.py                    │   │   └── b/
├── settings_a.py                  │   └── services/    # Background workers
├── settings_b.py                  ├── docs/            # Architecture docs
├── upload.py                      └── status/          # Agent workspace
├── config.py
└── ... (80 more files)

Agent reads 30 files               Agent goes directly
trying to find the                  to the right folder
relevant one
```

A `docs/` folder with numbered files gives the agent a reading order:

```
docs/
├── 00-OVERVIEW.md          # Start here — structure, tech stack, conventions
├── 01-ARCHITECTURE.md      # How the system is built
├── 02-DATA-FLOW.md         # How data moves through the system
├── 03-INSTALLATION.md      # How to set it up
├── ...
```

```
Without docs/                        With docs/

Agent explores filesystem:           Agent reads 00-OVERVIEW.md:
    │                                    │
    ├─ ls src/                           ├─ Sees full project map
    ├─ reads random files                ├─ Knows folder structure
    ├─ reads more files                  ├─ Knows tech stack
    ├─ reads even more                   ├─ Knows conventions
    ├─ pieces together a guess           └─ Navigates directly
    └─ maybe gets it right                   to the right file
                                    
    20 files read, 10 min.               1 file read, 30 sec.
```

Naming matters too:

```
Bad names:                           Good names:

docs/                                docs/
├── notes.md                         ├── 00-OVERVIEW.md
├── info.md                          ├── 01-ARCHITECTURE.md
├── guide.md                         ├── 02-DATA-FLOW.md
└── misc.md                          └── 03-INSTALLATION.md

src/                                 src/
├── stuff.py                         ├── base_module.py
├── things.py                        ├── base_settings.py
└── do.py                            └── json_helper.py

Which do you read first?             00 = start here.
No idea.                             base_ = parent class.
                                     Name = purpose.
```

The `00` file is the most important. It should contain: the project's purpose, the folder structure, the tech stack, the key conventions, and links to deeper docs.

---

## 4. Centralize shared code — duplication kills agent fixes

### The principle

Individual modules should be as thin as possible — just the logic specific to that module's job. All shared infrastructure lives in one shared package. Fix a core bug once in one file, not across twenty copies. When leaves are thin, the agent's context window goes further.

### The problem: Duplicated logic across files

If the same pattern is copy-pasted across 20 files, the agent doesn't know which copy is canonical. It fixes one and misses the rest.

```
module_a.py ──> def download():  ... (v1 of the logic)
module_b.py ──> def download():  ... (v1, slightly modified)
module_c.py ──> def download():  ... (v2, someone changed it)
module_d.py ──> def download():  ... (v1 again)
...
module_t.py ──> def download():  ... (v3, yet another variant)

Agent fixes the bug in module_a.
The other 19 copies? Still broken.
Which version is "correct"? Nobody knows.
```

### The solution: One shared package, thin modules

Put shared logic in one place. Individual modules only contain their unique logic.

```
Bad: Fat leaves                      Good: Thin leaves, fat core

module_a.py (300 lines)              libs/base.py (300 lines)
  ├─ parsing logic (30 lines)            ├─ download logic
  ├─ download logic (100 lines)          ├─ upload logic
  ├─ upload logic (100 lines)            ├─ error handling
  └─ error handling (70 lines)           └─ retry logic
                                              │
module_b.py (300 lines)              module_a.py (30 lines)
  ├─ parsing logic (30 lines)            └─ extends base.py
  ├─ download logic (100 lines)              └─ only parsing logic
  ├─ upload logic (100 lines)
  └─ error handling (70 lines)       module_b.py (30 lines)
                                         └─ extends base.py
Bug fix: 20 files to change.                 └─ only parsing logic
Agent fixes 1, misses 19.
                                     Bug fix: 1 file changed.
                                     All modules fixed.
```

**Bad:**

```python
# module_a.py
class WorkerA:
    def download(self, url):
        # 100 lines of download + retry + upload logic

# module_b.py
class WorkerB:
    def download(self, url):
        # Same 100 lines, slightly different
```

**Good:**

```python
# libs/base.py
class BaseWorker:
    def download(self, url):
        # 100 lines — lives here once

# module_a.py
from libs.base import BaseWorker
class WorkerA(BaseWorker):
    def parse(self, data):
        # Only module-specific logic — 20 lines
```

Agent fixes the download bug in `libs/base.py`. All 20 modules are fixed.

---

## 5. One concern per location — scattered info breaks fixes

### The principle

Every question an agent might ask should have exactly one place to look. If the agent has to search three places to answer one question, your structure has a problem.

### The problem: Same info in multiple places

When the same value, config, or concept is defined in multiple files, the agent finds one, changes it, and misses the others. The codebase ends up in an inconsistent state.

```
Bad: Same config in 4 places        Good: Single source of truth

  config.py ──> host = "10.0.0.1"       .env ──> DB_HOST=10.0.0.1
  module_a.py ──> host = "10.0.0.1"           │
  module_b.py ──> host = "10.0.0.2"  (!)      ▼
  docs/setup.md ──> host = 10.0.0.1      config("DB_HOST")
                                              │
  Agent changes one,                     ┌────┴─────┐
  misses the others.                     ▼          ▼
  Inconsistent state.               module_a    module_b

                                    Agent changes .env.
                                    Done.
```

### The solution: Single source of truth for everything

**Bad:** Database host is in `config.py`, also hardcoded in two modules with different values, and mentioned again in the docs. Agent fixes one, misses the others.

**Good:** Config is defined once in `.env`, read through `config()`, documented in one table. Agent changes one location and it's done.

This applies beyond config — it applies to any shared knowledge:

```
Bad:                                 Good:

"How do we handle auth?"             "How do we handle auth?"
  ├─ README says JWT                   └─ docs/01-ARCHITECTURE.md
  ├─ module_a uses sessions                says JWT + links to
  └─ docs/setup.md says cookies            src/libs/auth.py

  Three answers.                     One answer. One location.
  Agent picks one at random.         Agent reads it and knows.
```

---

## 6. Keep agents focused — they wander without structure

### The principle

Without clear boundaries, agents wander. Every file they read is a potential distraction. Conventions help — when every module follows the same pattern, there's nothing to "fix" and no reason to drift.

### The problem: Agent drift

You ask the agent to fix a bug in module A and it starts "improving" the logging in module B, refactoring imports in module C, and adding type hints everywhere — none of which you asked for.

```
You: "Fix the retry logic in module A"

Agent's actual path:
    │
    ├─ Reads module A           ← on task
    ├─ Reads module B           ← "while I'm here..."
    ├─ Refactors module B       ← off task
    ├─ Reads config.py          ← "I noticed..."
    ├─ Adds type hints          ← completely off task
    ├─ Updates imports           ← still off task
    └─ "Oh, about module A..."  ← finally back, context spent
```

### The solution: Structured dev plan + conventions

Two things keep the agent on track: a plan with explicit steps, and conventions that remove the temptation to "improve" things.

**The dev plan** (`status/DEV.md`) gives the agent a specific task with checkboxes. It knows what to do and when it's done. No room to wander.

**Bad:**

```
You: "There are some bugs in the codebase, can you take a look?"

Agent: *reads 15 files, refactors 3 of them, adds logging to 2 others,
       suggests a new folder structure, and eventually mentions one bug*
```

**Good:**

```markdown
## FIX-03: Fix timeout handling in module A
**Steps:**
- [ ] Add timeout parameter to download()
- [ ] Add retry with backoff on timeout
- [ ] Update config with default timeout value
```

```
Agent reads FIX-03:
    │
    ├─ Opens module A
    ├─ Adds timeout parameter         ← [x]
    ├─ Adds retry with backoff        ← [x]
    ├─ Updates config                 ← [x]
    └─ Done. Moves to FIX-04.
```

**Conventions** reduce drift further. When every module follows the same pattern, the agent doesn't feel the urge to "fix" differences — there are none.

```
Without conventions:                 With conventions:

module_a ──> class A(Custom):        module_a ──> class A(Base):
module_b ──> function_b():               parse()
module_c ──> class C(Other):         module_b ──> class B(Base):
module_d ──> async def d():              parse()
                                     module_c ──> class C(Base):
Agent sees inconsistency.                parse()
Starts "fixing" it.
Drifts off task.                     Nothing to "fix." Stays on task.
```

Document your conventions in `docs/`. If the agent doesn't know the expected pattern, it will improvise.

---

## 7. Prompting — how you talk to the agent matters

### The problem

The same task can take 2 minutes or 20 minutes depending on how you prompt the agent. Vague prompts produce vague work. The agent fills in the gaps with assumptions — often wrong ones.

### Bad prompts vs good prompts

**Bad:** Vague, open-ended, no boundaries.

```
"Fix the bugs in the codebase"

Agent: *reads 15 files, refactors 3, adds logging to 2,
       changes a naming convention, and eventually mentions one bug*
```

```
"Make the download faster"

Agent: *rewrites the entire module from scratch with a different
       library, breaks 4 other things in the process*
```

```
"Can you improve this code?"

Agent: *adds type hints, reformats everything, renames variables,
       changes the error handling strategy, and touches 8 files
       you didn't ask about*
```

**Good:** Specific, scoped, with context.

```
"Fix the retry logic in src/libs/http_client.py — it resets
the counter on redirect before checking for 429. The counter
should persist across redirects."

Agent: *opens the one file, fixes the one bug, done*
```

```
"Add a timeout parameter to the download() method in libs/base.py.
Default 30 seconds. Raise RetryableError on timeout so the existing
retry logic handles it."

Agent: *makes the exact change, respects the existing patterns*
```

```
"Read status/DEV.md, continue from FIX-03."

Agent: *reads the plan, sees the checkboxes, starts on the next
       unchecked step — no guessing needed*
```

### Prompting rules

```
┌─────────────────────────────────────────────────────┐
│                  GOOD PROMPTS                        │
│                                                      │
│  ✓ Name the specific file or module                  │
│  ✓ Describe the exact behavior you want              │
│  ✓ Mention what should NOT change                    │
│  ✓ Point to docs or status/ for context              │
│  ✓ One task per prompt                               │
│                                                      │
│                  BAD PROMPTS                          │
│                                                      │
│  ✗ "Fix the bugs" (which bugs? where?)               │
│  ✗ "Improve this" (improve how? by what measure?)    │
│  ✗ "Refactor the codebase" (all of it? to what?)     │
│  ✗ "Make it better" (the agent decides what "better" │
│     means — you won't like its definition)           │
│  ✗ Multiple unrelated tasks in one prompt            │
└─────────────────────────────────────────────────────┘
```

**The pattern:**

```
Bad:  "Fix the download logic"
       ↓
       Agent interprets "fix" however it wants.
       Touches 5 files. Breaks 2.

Good: "In libs/base.py, the download() method doesn't
       clean up temp files on failure. Add a finally
       block that deletes the temp file."
       ↓
       Agent makes one change in one file. Done.
```

The more specific you are, the less the agent guesses. The less it guesses, the fewer things break.

---

## 8. Use IDE-based agents, not web-based AI tools

### The problem with web-based AI (ChatGPT, Claude web, etc.)

When you paste code into a web-based AI tool, you're working backwards. You copy a file, paste it into the browser, explain the project structure in text, wait for a response, then manually copy the code back into your editor. The AI has no access to your codebase — it sees only the fragment you pasted.

```
Web-based AI workflow:

  Your codebase                    Browser
  ┌──────────┐     copy/paste     ┌──────────┐
  │ 80 files │ ──────────────>    │ 1 file   │
  │          │                    │ pasted   │
  │          │     copy/paste     │          │
  │          │ <──────────────    │ response │
  └──────────┘                    └──────────┘

  You are the middleware.
  You copy. You paste. You explain context manually.
  The AI sees a fragment. It guesses the rest.
```

### Why IDE-based agents (Cursor, Copilot, etc.) are better

An IDE-based agent sits inside your editor. It can read your files, search your codebase, see your folder structure, run commands, and edit code directly. You don't copy anything — the agent navigates the project itself.

```
IDE-based agent workflow:

  Your codebase
  ┌──────────────────────────┐
  │ 80 files                 │
  │   │                      │
  │   ├─ Agent reads files   │
  │   ├─ Agent searches code │
  │   ├─ Agent edits files   │
  │   ├─ Agent runs commands │
  │   └─ Agent checks lints  │
  │                          │
  │  Agent lives HERE.       │
  │  No copy/paste needed.   │
  └──────────────────────────┘
```

### The key advantages

```
                        Web AI          IDE Agent
                        ──────          ─────────
Sees your codebase       No ✗           Yes ✓
Reads any file           No ✗           Yes ✓
Searches across files    No ✗           Yes ✓
Edits files directly     No ✗           Yes ✓
Runs terminal commands   No ✗           Yes ✓
Knows folder structure   No ✗           Yes ✓
Switches models freely   No ✗           Yes ✓
Context from docs/       Manual ✗       Automatic ✓
```

**Switching models matters.** In an IDE agent like Cursor, you can use any model — switch between them based on the task. A fast model for simple edits, a stronger model for complex reasoning. The agent reads your `docs/` and `status/` regardless of which model is running. Web AI locks you to one model and one interface.

**No wait time on context.** The web AI workflow wastes time on copying, pasting, and explaining. The IDE agent already knows your project structure. Tell it "read `status/DEV.md` and continue" — it reads the file and starts working. No manual context transfer.

**Bad:** You open ChatGPT, paste 3 files, write a paragraph explaining the project structure, ask for a fix, get a response, manually apply the diff, realize it broke something in a file the AI never saw.

**Good:** You open Cursor, type "Fix the retry logic in `src/libs/http_client.py` — the counter resets on redirect." The agent reads the file, reads the imports, checks related files if needed, makes the edit, and you review the diff in place.

```
Web AI:     Copy → Paste → Explain → Wait → Copy back → Hope it fits
IDE Agent:  Prompt → Agent reads → Agent edits → You review → Done
```

Stop being the middleware between the AI and your code. Let the agent work where the code lives.

---

## The Workflow

```
┌──────────────────────────────────────────────────────────────┐
│                    SESSION LIFECYCLE                          │
│                                                              │
│  Start ──> Read status/DEV.md ──> Find first unchecked box   │
│                                          │                   │
│                                          ▼                   │
│                                    Work on task              │
│                                    Check boxes               │
│                                          │                   │
│                              ┌───────────┴──────────┐        │
│                              ▼                      ▼        │
│                        Task done?            Chat decaying?  │
│                              │                      │        │
│                              ▼                      ▼        │
│                     Mark [x] in DEV.md     Update docs       │
│                     Move to next task      Start fresh chat  │
│                                                              │
│  End ──> Checkboxes reflect progress ──> Next session ready  │
└──────────────────────────────────────────────────────────────┘
```

### Starting a new agent session

> "Read `status/DEV.md` and `status/BUGS.md`. Pick up from the first incomplete task."

That's it. The agent reads the plan, sees the checkboxes, and starts working.

### While the agent works

- The agent marks checkboxes in `DEV.md` as it completes steps.
- If it finds a new bug, it adds to `BUGS.md` and creates a task in `DEV.md`.
- One fix at a time. Finish it, test it, mark it done, then move on.

### When the session ends

- Check the last few boxes in `DEV.md` to confirm they're accurate.
- If the agent was mid-task, the unchecked boxes show exactly where to resume.
- The next session can be with a completely different agent or model — the context is in the files.

### When adding a new feature

1. Write the plan in `DEV.md` with clear implementation steps and checkboxes.
2. If it touches architecture, update the relevant `docs/` file.
3. Hand it to the agent. The agent follows the plan, checks boxes, and you review.

### When onboarding a new developer (human or AI)

1. `README.md` — project summary
2. `docs/00-OVERVIEW.md` — full architecture map
3. `status/` — current state of development

Three reads and they're productive.

---

## Quick Reference

| Problem | Solution |
|---------|----------|
| Session amnesia | `status/` folder with `BUGS.md` and `DEV.md` as persistent memory |
| Context decay | Document every strategy, rotate to fresh chats freely |
| Flat structure / context blindness | Organized hierarchy + numbered `docs/` + descriptive names |
| Duplicated logic | Centralized shared library — fix once, fixed everywhere |
| Scattered info | Single source of truth — one location per concern |
| Agent drift | Structured dev plan with checkboxes + documented conventions |
| Vague output / wasted tokens | Specific prompts — name the file, describe the behavior, one task |
| Manual copy-paste / no codebase access | IDE-based agents (Cursor, etc.) — agent lives in the codebase |

The codebase is the agent's brain. Structure it well, and every agent session starts smart.
