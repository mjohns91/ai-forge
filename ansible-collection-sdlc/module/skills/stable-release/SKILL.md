---
name: stable-release
description: >-
  Complete end-to-end release workflow orchestrator for Ansible collections
  using stable-X branch strategy. Coordinates stable-release-analyze, 
  stable-release-prep, docs-generate, lint, and sanity skills for a fully
  automated stable branch release process. Use when the user wants to release
  a collection with multiple stable branches.
imports:
  - stable-release-analyze
  - stable-release-prep
  - docs-generate
  - tox-lint
  - sanity
---

# Stable Branch Release Orchestrator

Complete end-to-end release workflow for collections with stable-X branches.

## Author

**Alina Buzachis (alinabuzachis)** - Ansible Cloud Content Team

## Reference Documentation

- Ansible Cloud Content Handbook: https://github.com/ansible-collections/cloud-content-handbook
- Release Process: https://github.com/ansible-collections/cloud-content-handbook/blob/main/stable-release.md

## Purpose

Orchestrates all release steps from analysis to PR creation for stable-branch workflows. Imports
and coordinates individual skills (stable-release-analyze, stable-release-prep, docs-generate,
tox-lint, sanity) in the correct order with proper error handling and user interaction.

## File Operations - Batch Processing

**CRITICAL**: This orchestrator must minimize permission prompts by batching operations.

**Key principles**:

1. **Virtual environment first**: Set up .venv with all required dependencies (antsibull-changelog, ansible-core, tox)
2. **Batch all file reads**: Read galaxy.yml, changelog fragments, and other files in ONE message with parallel tool calls
3. **Parallel quality checks**: Run `/tox-lint` and `/sanity` in parallel using background agents
4. **Sequential git operations**: Git operations via Bash tool are naturally batched in commands

**Example flow**:

```
Message 1: Setup venv with uv/pip (antsibull-changelog, ansible-core, tox)
Message 2: Read galaxy.yml + all changelog fragments (parallel)
Message 3: Run git operations (single bash command with &&)
Message 4: Invoke /tox-lint and /sanity (parallel, run_in_background)
Message 5: Write/Edit files after analysis complete
```

**Virtual environment setup** (use uv for speed):

```bash
python3 -m venv .venv && source .venv/bin/activate && \
(command -v uv &> /dev/null && \
  uv pip install antsibull-changelog ansible-core tox || \
  pip install --quiet antsibull-changelog ansible-core tox)
```

## Configuration

Reads from `~/.ansible-release.conf`:

```bash
export GITHUB_USERNAME="alinabuzachis"
export ANSIBLE_COLLECTIONS_PATH="~/dev/collections/ansible_collections"
export SANITY_MODE="smart"
export AUTO_CREATE_PR="prompt"  # true | false | prompt
export LINT_ON_COMMIT="true"
export SANITY_ON_COMMIT="true"
```

## When to Use This Skill

- Performing a complete collection release
- Automating the release process from start to finish
- After changelog fragments have been created
- When asked to release, publish, or ship a new version
- To follow the cloud-content-handbook release process

## Usage Examples

```bash
# Full automated release (interactive prompts)
/stable-release

# Analyze only (no changes)
/stable-release --analyze-only

# Prepare specific version
/stable-release --collection amazon.ai --version 1.0.1 --branch stable-1

# Custom release date (future-dated or backdated)
/stable-release --collection amazon.ai --version 1.0.1 --branch stable-1 --release-date 2026-06-15

# Skip specific steps
/stable-release --skip-lint --skip-sanity

# Full automation (no prompts)
/stable-release --auto --create-pr

# Dry run (show what would happen)
/stable-release --dry-run
```

## Workflow Steps

The orchestrator executes skills in this order:

### Step 1: Analyze (`stable-release-analyze`)

**Purpose**: Determine if release is needed and calculate version

```bash
/stable-release-analyze ${COLLECTION}
```

- Checks stable branches for unreleased commits
- Analyzes changelog fragments
- Calculates SemVer bump (major/minor/patch)
- Generates release plan

**Output**:

```
Branch: stable-1
Current: 1.0.0 → Proposed: 1.0.1 (PATCH)
Reason: 1 bugfix fragment
```

**Prompt** (unless `--auto`):

```
Found pending release for stable-1 → v1.0.1
Proceed with release? [Y/n]:
```

If `--analyze-only`: Stop here and exit.

### Step 2: Prepare (`stable-release-prep`)

**Purpose**: Create release branch and update files

```bash
/stable-release-prep --collection ${COLLECTION} --version ${VERSION} --branch ${BRANCH} \
  ${RELEASE_DATE:+--release-date ${RELEASE_DATE}}
```

- Creates `prep_vX.Y.Z` branch
- Updates `galaxy.yml` version
- Creates release summary fragment (with \`\`backticks\`\`)
- Runs `antsibull-changelog release` (with custom date if --release-date provided)

**Output**:

```
✅ Branch created: prep_v1.0.1
✅ galaxy.yml updated: 1.0.0 → 1.0.1
✅ Changelog generated
```

**Prompt** (unless `--auto`):

```
Review git diff before proceeding? [Y/n]:
```

If yes: Display `git diff --stat`

### Step 3: Documentation (`docs-generate`)

**Purpose**: Update module documentation

```bash
/docs-generate
```

- Runs `tox -e add_docs`
- Updates module RST files
- Updates README.md

**Output**:

```
✅ Documentation updated (12 files)
```

Skip if `--skip-docs` flag provided.

### Step 4: Quality Checks (Parallel)

**Purpose**: Validate code quality and tests

**CRITICAL**: Execute in parallel using a SINGLE message with TWO Skill tool calls:

```
In ONE message, invoke:
  - Skill: "tox-lint" with args "--path=${COLLECTION_PATH}"
  - Skill: "sanity" with args "--mode=${SANITY_MODE} --path=${COLLECTION_PATH}"
  
This batches permission requests and runs checks in parallel.
```

Alternative using Agent tool (if Skill doesn't support parallel):

```bash
# Launch both concurrently in same message
Agent(skill="tox-lint", args="--path=${COLLECTION_PATH}", run_in_background=True)
Agent(skill="sanity", args="--mode=${SANITY_MODE} --path=${COLLECTION_PATH}", run_in_background=True)

# Claude Code will notify when both complete
```

**Output**:

```
Running quality checks...
  /tox-lint --path=${COLLECTION_PATH} (parallel)
  /sanity --mode=smart --path=${COLLECTION_PATH} (parallel)

✅ tox-lint: All checks passed (84.7s)
✅ sanity: All tests passed (45.3s)
```

**On Failure**:

```
❌ Quality checks failed

tox-lint: FAILED
  - black-lint: 3 files need formatting

sanity: FAILED
  - validate-modules: Missing RETURN docs in devopsguru_resource_collection.py

Options:
  [r] Retry after fixing
  [s] Skip checks (not recommended)
  [a] Abort

Choice [r/s/a]:
```

Skip if `--skip-lint` or `--skip-sanity` flags provided.
Continue despite failures if `--force` flag provided.

### Step 5: Commit & Push (`release-commit-push`)

**Purpose**: Commit changes and push to fork

```bash
git commit and git push
```

- Stages release files
- Commits with standard message
- Pushes to origin (fork)

**Output**:

```
✅ Committed: 46a848a
✅ Pushed to origin/prep_v1.0.1
```

### Step 6: Pull Request (Conditional)

**Purpose**: Create PR to upstream

Based on `AUTO_CREATE_PR` config or `--create-pr` flag:

**If `AUTO_CREATE_PR=prompt`** (default):

```
Create pull request now? [Y/n]:
```

**If yes or `AUTO_CREATE_PR=true`**:

```bash
gh pr create \
  --repo ${UPSTREAM_ORG}/${REPO} \
  --base ${BRANCH} \
  --head ${GITHUB_USERNAME}:prep_v${VERSION} \
  --title "Release v${VERSION}" \
  --body "$(cat <<EOF
## Release v${VERSION}

This PR prepares the v${VERSION} release for the ${NAMESPACE}.${NAME} collection.

### Changes
- Updated galaxy.yml to v${VERSION}
- Generated changelog from fragments
- Updated documentation

### Quality Checks
- ✅ Lint: All checks passed
- ✅ Sanity: All tests passed

### Checklist
- [x] Version updated in galaxy.yml
- [x] Changelog generated
- [x] Documentation updated
- [x] Lint checks passed
- [x] Sanity tests passed

---
*Generated by Ansible Release Orchestrator*
*Reference: https://github.com/ansible-collections/cloud-content-handbook*
EOF
)"
```

**Output**:

```
✅ PR created: https://github.com/ansible-collections/amazon.ai/pull/42
```

## Complete Output Example

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Ansible Collection Release Orchestrator
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Collection: amazon.ai
Working directory: /Users/alinabuzachis/dev/collections/ansible_collections/amazon/ai
Reference: https://github.com/ansible-collections/cloud-content-handbook

[1/6] Analyzing pending releases...
      Running: /stable-release-analyze

      ✅ Analysis complete
      Branch: stable-1
      Current: 1.0.0 → Proposed: 1.0.1 (PATCH)
      Reason: 1 bugfix fragment

      Proceed with release v1.0.1? [Y/n]: y

[2/6] Preparing release branch...
      Running: /stable-release-prep --version 1.0.1 --branch stable-1

      ✅ Branch created: prep_v1.0.1
      ✅ galaxy.yml updated
      ✅ Changelog generated

[3/6] Generating documentation...
      Running: /docs-generate

      ✅ Documentation updated (12 files)

[4/6] Running quality checks...
      Running: /tox-lint --path=${COLLECTION_PATH} (parallel)
      Running: /sanity --mode=smart --path=${COLLECTION_PATH} (parallel)

      ✅ tox-lint: All checks passed (84.7s)
      ✅ sanity: All tests passed (45.3s)

[5/6] Committing and pushing...
      Running: git commit and git push

      ✅ Committed: 46a848a
      ✅ Pushed to origin/prep_v1.0.1

[6/6] Creating pull request...
      Auto-create PR: prompt

      Create PR now? [Y/n]: y

      ✅ PR created: https://github.com/ansible-collections/amazon.ai/pull/42

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Release workflow complete! 🎉
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Summary:
  Collection: amazon.ai
  Version: 1.0.0 → 1.0.1
  Branch: prep_v1.0.1
  Commit: 46a848ad98855647d56e00bcbb5b5fd4b8ce67f7
  PR: https://github.com/ansible-collections/amazon.ai/pull/42

Next steps:
  1. Monitor PR: https://github.com/ansible-collections/amazon.ai/pull/42
  2. After merge, tag release: git tag v1.0.1 && git push upstream v1.0.1
  3. GitHub Actions will publish to Galaxy

Total time: 2m 34s
```

## Error Handling & Recovery

### Recoverable Errors

**Lint/Sanity Failures**:

```
Options:
  [r] Retry after fixing issues
  [s] Skip this step (--force)
  [a] Abort workflow

Choice [r/s/a]:
```

**Merge Conflicts**:

```
⚠️ Merge conflict detected while syncing stable-1

Actions:
  1. Resolve conflicts manually
  2. Run: git add <files>
  3. Re-run: /stable-release --resume

Abort workflow? [y/N]:
```

**Network Errors**:

```
⚠️ Failed to push to origin (network error)

Retrying in 5 seconds... (attempt 2/3)
```

### Non-Recoverable Errors

**Missing Configuration**:

```
❌ Error: GITHUB_USERNAME not set

Setup required:
  1. Create config: cp ansible-release.conf.template ~/.ansible-release.conf
  2. Edit config: vim ~/.ansible-release.conf
  3. Set GITHUB_USERNAME="your-username"
  4. Re-run: /stable-release
```

**Not a Git Repository**:

```
❌ Error: Not a git repository

Ensure you're in a collection directory:
  cd ~/dev/collections/ansible_collections/namespace/collection
```

### Resume Capability

State is tracked in `.ansible-release-state.json`:

```json
{
  "collection": "amazon.ai",
  "version": "1.0.1",
  "branch": "prep_v1.0.1",
  "last_completed_step": "docs-generate",
  "timestamp": "2026-04-01T14:45:00Z"
}
```

Resume workflow:

```bash
/stable-release --resume
```

Output:

```
Resuming from last checkpoint...
Last completed: docs-generate
Continuing with: quality checks
```

## Flags & Options

| Flag | Description |
| ----- | ----------- |
| `--analyze-only` | Run analysis and stop |
| `--release-date YYYY-MM-DD` | Custom release date (default: today) |
| `--skip-lint` | Skip linting step |
| `--skip-sanity` | Skip sanity testing |
| `--skip-docs` | Skip documentation generation |
| `--force` | Continue even if quality checks fail |
| `--auto` | No interactive prompts (use config defaults) |
| `--create-pr` | Automatically create PR |
| `--dry-run` | Show what would happen without making changes |
| `--resume` | Resume from last successful step |

## Integration Points

This orchestrator imports:

- `/stable-release-analyze` - Determine version needed
- `/stable-release-prep` - Create branch and changelog
- `/docs-generate` - Update documentation
- `/tox-lint` - Run linters
- `/sanity` - Run sanity tests
- `git commit and git push` - Commit and push

## Requirements

### System Requirements

- `git` with configured remotes
- `tox` with lint and add_docs environments
- `antsibull-changelog` installed
- `gh` CLI (for PR creation)

### Repository Requirements

- Git remotes: `origin` (fork), `upstream` (canonical)
- `galaxy.yml` with valid version
- `tox.ini` with lint and add_docs envs
- `changelogs/fragments/` with at least one fragment

### Configuration Requirements

- `~/.ansible-release.conf` with GITHUB_USERNAME
- SSH keys or HTTPS credentials for GitHub

## Exit Codes

- `0`: Release workflow completed successfully
- `1`: Workflow failed at any step
- `2`: Configuration error or missing requirements
- `130`: User aborted (Ctrl+C)

## Implementation Notes

### Parallel Execution

Run lint and sanity concurrently for speed:

```python
# Use Claude Code's ability to run agents in parallel
Agent(skill="tox-lint", args="--path=${COLLECTION_PATH}", run_in_background=True)
Agent(skill="sanity", args="--mode=${SANITY_MODE} --path=${COLLECTION_PATH}", run_in_background=True)

# Claude Code will notify when both complete
# Then proceed to next step
```

### State Management

```python
import json
from pathlib import Path
from datetime import datetime

STATE_FILE = Path.cwd() / ".ansible-release-state.json"

def save_state(step: str, data: dict):
    state = {
        "last_completed_step": step,
        "timestamp": datetime.now().isoformat(),
        **data
    }
    STATE_FILE.write_text(json.dumps(state, indent=2))

def load_state():
    if STATE_FILE.exists():
        return json.loads(STATE_FILE.read_text())
    return None

def clear_state():
    if STATE_FILE.exists():
        STATE_FILE.unlink()
```

### User Interaction

Provide clear prompts and progress:

- Show step numbers [1/6], [2/6], etc.
- Use status indicators: ✅ ❌ ⚠️
- Provide actionable error messages
- Offer resume capability on failure
- Display total time at completion

### Cloud Content Handbook Compliance

Follow handbook guidelines:

- Use double backticks for module names
- Standard commit message format
- Co-Authored-By: Claude attribution
- PR template with checklist
- Quality checks before commit
