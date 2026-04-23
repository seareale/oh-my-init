---
name: smart-commit
description: Use when committing changes — groups related changes into logical commits, runs pre-commit hooks, auto-fixes lint issues, writes conventional commit messages with module scope, and pushes
---

# Smart Commit

## Overview

Analyze uncommitted changes, group them into logical commits, run pre-commit hooks with auto-fix loops, write conventional commit messages, and push.

**Core principle:** Each commit is one logical unit. Pre-commit must pass before committing. Auto-fix everything fixable.

**Announce at start:** "I'm using the smart-commit skill to commit and push changes."

## The Process

### Step 0: Prerequisites

Detect the Python runner and ensure required tools are installed.

#### 0a. Detect Runner

```bash
if command -v uv &>/dev/null; then
  RUN="uv run"
else
  RUN="python -m"
fi
```

Use `$RUN` as the prefix for all Python tool commands throughout this process.

#### 0b. Check & Install Dependencies

Verify `pre-commit`, `pytest`, and `pytest-cov` are available:

```bash
$RUN pre-commit --version 2>/dev/null || { echo "Installing pre-commit..."; pip install pre-commit; }
$RUN pytest --version 2>/dev/null || { echo "Installing pytest..."; pip install pytest; }
python -c "import pytest_cov" 2>/dev/null || { echo "Installing pytest-cov..."; pip install pytest-cov; }
```

If `uv` is available, prefer `uv pip install` instead of `pip install`.

### Step 1: Analyze Changes

```bash
git --no-pager diff --stat HEAD
git --no-pager diff --stat --cached HEAD
git --no-pager status --short
```

Read the output. Understand what changed and why.

### Step 2: Group Changes Into Logical Commits

Group related files into commit units based on:

1. **Functional cohesion** — files that implement the same feature/fix together
2. **Module boundary** — changes within the same module/package
3. **Type of change** — separate `feat`, `fix`, `refactor`, `test`, `docs`, `chore` changes

**Grouping rules:**
- A test file and its corresponding source change go in the same commit
- Config changes (pyproject.toml, setup.cfg) go with the feature that needs them
- Pure formatting/lint fixes are a separate commit (`style:`)
- Don't create too many tiny commits — if 3 files all serve one purpose, group them
- Don't create one massive commit either — separate distinct logical changes

**Present the grouping plan to the user before proceeding:**

```
I've analyzed the changes and propose these commits:

1. feat(evaluator): add evaluator tests
   - tests/evaluator/test_evaluators.py (new)

2. fix(benchmark): comment out debug skip code
   - src/benchmark.py

3. ...

Proceed with this grouping?
```

Use `ask_user` tool to confirm. If the user suggests changes, adjust.

### Step 3: For Each Commit Group — Stage & Pre-commit Loop

Process each group sequentially:

#### 3a. Stage Files

```bash
git add <file1> <file2> ...
```

#### 3b. Run Pre-commit on Staged Files

```bash
$RUN pre-commit run --files <file1> <file2> ...
```

**Do NOT use `--all-files`** — only run on the files being committed.

#### 3c. Interpret Pre-commit Output

Pre-commit hooks in this project:
- `end-of-file-fixer` — auto-fixes (adds trailing newline)
- `trailing-whitespace` — auto-fixes (removes trailing spaces)
- `check-yaml` — fails on invalid YAML (must fix manually)
- `check-json` — fails on invalid JSON (must fix manually)
- `black` — auto-fixes (reformats Python code)
- `isort` — auto-fixes (reorders imports)
- `flake8` — reports errors only (must fix manually)

**Auto-fix hooks** (`black`, `isort`, `end-of-file-fixer`, `trailing-whitespace`):
- These modify files in-place on failure
- After they run, the files are modified but NOT re-staged
- You MUST re-stage: `git add <modified files>`

**Report-only hooks** (`flake8`, `check-yaml`, `check-json`):
- These report errors without fixing
- You must read the error, fix the code, and re-stage

#### 3d. Run Tests

**Skip this step if the project has no `tests/` directory.**

Detect the coverage target automatically:

```bash
if [[ -d tests ]]; then
  # Auto-detect --cov paths: all package directories under src/
  COV_TARGETS=$(find src -mindepth 1 -maxdepth 1 -type d ! -name '__pycache__' ! -name '*.egg-info' 2>/dev/null)
  if [[ -z "$COV_TARGETS" ]]; then
    # Fallback: use project name from pyproject.toml
    COV_TARGETS=$(grep -Po '(?<=name = ")[^"]+' pyproject.toml | head -1 | tr '-' '_')
  fi

  if [[ -n "$COV_TARGETS" ]]; then
    # Build --cov flags for each package
    COV_FLAGS=$(echo "$COV_TARGETS" | xargs -I{} echo "--cov={}" | tr '\n' ' ')
    $RUN pytest $COV_FLAGS --cov-report=term tests
  else
    $RUN pytest tests
  fi
fi
```

If tests fail, fix the code, re-stage, and re-run both pre-commit and pytest.

#### 3e. Fix Loop

```
while pre-commit OR pytest fails:
    1. If auto-fix hooks modified files → re-stage modified files
    2. If report-only hooks failed → read errors, fix code, re-stage
    3. If pytest failed → read errors, fix code, re-stage
    4. Run pre-commit again on the same files
    5. Run pytest again
```

**Max iterations: 5.** If still failing after 5 loops:
1. Display the last pre-commit and/or pytest error output to the user
2. Unstage all files: `git reset HEAD`
3. **Stop the entire smart-commit process** — do NOT proceed to remaining commit groups or push
4. Report: "Smart commit aborted after 5 fix attempts. Please resolve the errors manually."

#### 3f. Verify Clean State

After pre-commit and pytest both pass:
```bash
git --no-pager diff --cached --stat  # confirm staged files are correct
```

### Step 4: Write Commit Message

**Format:** Conventional Commits with module scope

```
<type>(<module>): <description>

<optional body>

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
```

**Types:**
| Type | When to use |
|------|-------------|
| `feat` | New feature, new capability |
| `fix` | Bug fix |
| `refactor` | Code restructuring, no behavior change |
| `test` | Adding or fixing tests |
| `docs` | Documentation only |
| `style` | Formatting, lint fixes (no logic change) |
| `chore` | Build process, dependencies, tooling |
| `perf` | Performance improvement |


**Description rules:**
- Lowercase, no period at end
- Imperative mood ("add" not "added")
- Under 72 characters
- Describe WHAT changed, not HOW

**Body (optional):**
- Add when the change is non-obvious
- List key changes as bullet points
- Keep it brief — the diff tells the full story

**Examples:**

```
test(evaluator): add comprehensive domain evaluator tests

- 295 tests covering all 20 domain evaluator classes
- Mock-based testing for dataset loading and inference
- Edge case coverage for empty results, missing fields

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
```

```
fix(benchmark): disable debug skip in sequential benchmark

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
```

### Step 5: Commit

```bash
git commit -m "<message>"
```

### Step 6: Repeat for Remaining Groups

Go back to Step 3 for the next commit group. Process all groups sequentially.

### Step 7: Push

After all commits are done, check if local and remote are in sync:

```bash
git fetch origin
local=$(git rev-parse HEAD)
remote=$(git rev-parse origin/$(git branch --show-current) 2>/dev/null || echo "none")
```

- If `remote` is an ancestor of `local` (fast-forward possible) → `git push`
- If `remote` is `none` (new branch) → `git push -u origin $(git branch --show-current)`
- Otherwise (diverged) → `git push --force`

## Quick Reference

```
1. git status + git diff --stat         → understand changes
2. Group files into logical commits      → present plan to user
3. For each group:
   a. git add <files>
   b. $RUN pre-commit run --files <files>
   c. (if tests/ exists) $RUN pytest --cov=<auto-detected> --cov-report=term tests
   d. Fix issues (pre-commit + pytest), re-stage, repeat until clean
   e. git commit -m "type(module): description"
4. git push (or git push --force if diverged)
```

## Common Mistakes

**Committing without running pre-commit**
- Problem: CI fails, commit hooks reject on push
- Fix: Always run `$RUN pre-commit run --files` before `git commit`

**Running pre-commit on all files**
- Problem: Fixes/errors in unrelated files, noisy output
- Fix: Always use `--files` flag with only the staged files

**Not re-staging after auto-fix**
- Problem: `black`/`isort` fix files but changes aren't staged → commit has old version
- Fix: After auto-fix hooks modify files, `git add` them again before re-running

**One giant commit for everything**
- Problem: Hard to review, hard to revert, messy history
- Fix: Group by logical unit — one feature/fix/refactor per commit

**Vague commit messages**
- Problem: "update files", "fix stuff", "wip"
- Fix: Follow conventional commits with module scope and clear description

**Forgetting Co-authored-by trailer**
- Problem: Missing attribution
- Fix: Always append the Copilot co-author trailer

## Red Flags

**Never:**
- Commit with failing pre-commit hooks or pytest
- Stage files from different logical changes in one commit
- Write commit messages that don't follow conventional format
- Skip the grouping confirmation with the user

**Always:**
- Analyze all changes before starting
- Present grouping plan and get confirmation
- Run pre-commit per commit group
- Re-stage after auto-fix hooks
- Include `Co-authored-by` trailer
- Push after all commits

## Integration

**Called by:**
- Any workflow that needs to commit and push changes
- **finishing-a-development-branch** — before merge/PR

**Pairs with:**
- **verification-before-completion** — verify tests pass before committing
- **requesting-code-review** — review changes before committing
