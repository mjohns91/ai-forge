---
name: aws-terminator-workflow
description: Complete end-to-end workflow for aws-terminator PR creation - analyze, implement, test, and submit
allowed-tools: Skill(skill:aws-terminator-analyze), Skill(skill:aws-terminator-implement), Read, Write, Bash(command:git *), Bash(command:gh *), Bash(command:python3 *)
argument-hint: "--pr <number> [--repo <owner/repo>] [--auto-pr] [--skip-tests] [--resume]"
triggers:
  - "aws terminator workflow"
  - "create aws-terminator pr"
  - "terminator pr for"
  - "aws-terminator workflow"
---

# AWS Terminator Workflow

Complete orchestrator for creating aws-terminator PRs when new AWS modules are added to Ansible collections. Coordinates analysis, implementation, testing, and PR submission.

## Purpose

When a PR adds new AWS modules to an Ansible collection (amazon.aws, community.aws, etc.), this workflow:

1. Analyzes what terminators and permissions are needed
2. Implements the terminator classes and IAM permissions
3. Runs validation tests
4. Creates a PR to mattclay/aws-terminator
5. Links it back to the original Ansible collection PR

## Quick Start

```bash
# Full automated workflow
/aws-terminator-workflow --pr 2353 --repo ansible-collections/community.aws

# Auto-mode (no prompts, fully automated)
/aws-terminator-workflow --pr 2353 --auto-pr
```

**What it does**: Analyzes the collection PR, implements terminators and IAM permissions in your fork, runs tests, creates a PR to mattclay/aws-terminator, and links it back to the original PR.

**Prerequisites**: Fork mattclay/aws-terminator on GitHub first. The skill handles cloning, branching, implementation, testing, and PR creation.

**See full documentation below** for detailed workflow steps, configuration options, and error recovery.

## When to Use

- Complete automation: "Create aws-terminator PR for community.aws#2353"
- After seeing CI failures due to missing permissions
- When reviewing a PR that adds new AWS modules
- End-to-end from analysis to PR submission

## Usage

```bash
# Full automated workflow
/aws-terminator-workflow --pr 2353 --repo ansible-collections/community.aws

# Default to community.aws
/aws-terminator-workflow --pr 2353

# Auto-mode (no prompts, automatic decisions)
/aws-terminator-workflow --pr 2353 --auto-pr

# Skip tox tests (faster, use for drafts)
/aws-terminator-workflow --pr 2353 --skip-tests

# Check mode (analyze only, don't implement)
/aws-terminator-workflow --pr 2353 --check

# Resume from failed/interrupted workflow
/aws-terminator-workflow --pr 2353 --resume
```

## Workflow Steps

### Step 1: Analyze the Ansible Collection PR

**Run analysis skill**:

```bash
/aws-terminator-analyze --pr <PR_NUMBER> --repo <REPO>
```

**Output**: Analysis report with:

- Resources being added
- Terminator coverage status
- IAM permissions needed
- Implementation recommendations

**Checkpoint**: If `--check` mode, stop here and present analysis only.

**Prompt** (unless `--auto-pr`):

```
Analysis complete. Found:
- N resource types need terminators
- M IAM permissions need to be added

Proceed with implementation? [Y/n]:
```

### Step 2: Check for Existing Terminator PR

**Search for related PRs**:

```bash
# Check if PR description mentions existing terminator PR
gh pr view <PR_NUMBER> --repo <REPO> --json body | \
  grep -o "mattclay/aws-terminator/pull/[0-9]*"
```

**If existing terminator PR found**:

**Prompt** (unless `--auto-pr`):

```
Found existing terminator PR: #<TERMINATOR_PR_NUMBER>
Status: <open/merged/closed>

Options:
  [u] Update existing PR
  [n] Create new PR
  [s] Skip implementation

Choice [u/n/s]:
```

**If merged**: Exit with message "Terminator PR already merged"
**If closed**: Warn and offer to create new PR
**If open**: Offer to update the existing PR

### Step 3: Setup aws-terminator Repository

**Clone or update**:

```bash
if [ ! -d ~/dev/aws-terminator ]; then
  echo "Cloning aws-terminator fork..."
  # Clone from YOUR fork, not mattclay's repo
  FORK_USER=$(gh api user --jq .login)
  git clone https://github.com/$FORK_USER/aws-terminator.git ~/dev/aws-terminator
  
  cd ~/dev/aws-terminator
  # Set up upstream remote to mattclay's repo
  git remote add upstream https://github.com/mattclay/aws-terminator.git
  git fetch upstream
fi

cd ~/dev/aws-terminator
# Sync your fork's main with upstream
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
```

**Create implementation branch**:

```bash
# Extract service name from analysis
SERVICE_NAME=$(echo "<first-resource-type>" | sed 's/[A-Z]/ &/g' | tr '[:upper:]' '[:lower:]' | awk '{print $1}')

# Branch naming: add-<service>-terminators
# Example: add-medialive-terminators
BRANCH_NAME="add-${SERVICE_NAME}-terminators"

git checkout -b "$BRANCH_NAME"
```

### Step 4: Implement Terminators and Permissions

**Run implementation skill**:

```bash
/aws-terminator-implement
```

This uses context from the analysis in Step 1.

**Implementation performs**:

- Creates terminator classes in appropriate `aws/terminator/*.py` files
- Adds IAM permissions to appropriate `aws/policy/*.yaml` files
- Follows aws-terminator code patterns
- Validates syntax

**Prompt** (unless `--auto-pr`) after each resource:

```
Implemented <ResourceType>Terminator in <file>.py
Added <N> IAM permissions to <policy-file>.yaml

Continue with next resource? [Y/n]:
```

### Step 5: Validation

**Python syntax check**:

```bash
cd ~/dev/aws-terminator

# Check all modified Python files
for file in $(git diff --name-only | grep '\.py$'); do
  python3 -m py_compile "$file" || echo "Syntax error in $file"
done
```

**YAML syntax check**:

```bash
# Check all modified YAML files
for file in $(git diff --name-only | grep '\.yaml$'); do
  python3 -c "import yaml; yaml.safe_load(open('$file'))" || echo "YAML error in $file"
done
```

**Run tox tests** (unless `--skip-tests`):

```bash
cd ~/dev/aws-terminator
tox
```

**Expected output**:

```
pycodestyle: OK
pylint: OK
yamllint: OK
policy: OK
congratulations :)
```

**On tox failure**:

**Prompt** (unless `--auto-pr`):

```
Tox validation failed:
<error output>

Options:
  [f] Fix and retry
  [s] Skip tox (continue anyway)
  [a] Abort workflow

Choice [f/s/a]:
```

### Step 6: Generate PR Description

**Create comprehensive PR body**:

````markdown
# Add <service> terminators and permissions

This PR adds comprehensive support for AWS <ServiceName> resources created by the Ansible <collection> collection.

## Related Ansible Collection PR

ansible-collections/<collection>#<PR_NUMBER>: <PR_TITLE>

## Changes

### Terminator Classes Added

**File**: `aws/terminator/<terminator-file>.py`

<For each terminator class>
- **<ResourceType>Terminator** - Handles <resource> lifecycle
  - Base class: `Terminator` | `DbTerminator`
  - List operation: `client.<list_operation>()`
  - Delete operation: `client.<delete_operation>()`
  - Special handling: [None | Pagination | Pre-delete stop | Child dependencies]

### IAM Permissions Added

**File**: `aws/policy/<policy-file>.yaml`

<For each permission block>
- **<ServiceName><ResourceType>Permissions** - Resource-scoped actions
  - Actions: <service>:Create*, Delete*, Describe*, Update*
  - Resources: `arn:aws:<service>:region:account:<resource-type>/*`

- **<ServiceName>GlobalPermissions** - List/Describe actions
  - Actions: <service>:List*, Describe*
  - Resources: `*`

## Testing

### Tox Validation

- ✅ pycodestyle: Passed
- ✅ pylint: Passed
- ✅ yamllint: Passed
- ✅ policy: Passed

### Manual Testing

<If manual testing was performed>
```bash
# Created test resources in AWS account
# Ran terminator in check mode
python cleanup.py --stage dev --target <ResourceType>Terminator -v -c

# Output:
cleanup: DEBUG located <ResourceType>Terminator: count=X
cleanup: DEBUG checked <ResourceType>Terminator: stale=True
```

## Implementation Notes

<Any special considerations>
- Child resources (e.g., <ChildType>) are auto-deleted with parent → no separate terminator needed
- <ResourceType> requires stop-before-delete logic
- Pagination required for <ListOperation> (can exceed default limit)

## Deployment Checklist

After merge:
- [ ] Deploy permissions: `make test_policy STAGE=dev`
- [ ] Test with ansible-test: `ansible-test integration <module> --remote-stage dev`
- [ ] Deploy to prod: `make test_policy STAGE=prod`
- [ ] Deploy lambda (if terminator classes changed): `make terminator_lambda`

## References

- Analysis: [Include analysis output or link]
- Ansible collection PR: https://github.com/ansible-collections/<collection>/pull/<PR_NUMBER>
- AWS <ServiceName> API docs: https://docs.aws.amazon.com/...
````

### Step 7: Finalize (Conditional on `--auto-pr` Flag)

**Without `--auto-pr` (default)**: Stop here and hand off to user

Output:

````markdown
## Implementation Complete ✅

**Status**: Ready to commit

### What Was Done

1. ✅ Analyzed ansible collection PR
2. ✅ Implemented <N> terminator classes in `aws/terminator/<terminator-file>.py`
3. ✅ Added <M> IAM permission blocks in `aws/policy/<policy-file>.yaml`
4. ✅ Validated implementation with tox tests
5. ✅ Generated comprehensive PR description (saved to `pr-description.md`)

### Files Modified

```bash
cd ~/dev/aws-terminator
git status
# Shows: aws/terminator/<file>.py, aws/policy/<file>.yaml
```

### Next Steps for You 👤

Review, commit, push, and create the PR manually:

#### 1. Review the Changes

```bash
git diff
```

#### 2. Commit

```bash
git add aws/terminator/<terminator-file>.py aws/policy/<policy-file>.yaml
git commit -m "Add <service> terminators and permissions

- Add <ResourceType>Terminator class
- Add IAM permissions for <service> operations

Required for ansible-collections/<collection> PR #<PR_NUMBER>"
```

#### 3. Push to Your Fork

```bash
git push origin add-<service>-terminators
```

#### 4. Create Pull Request

```bash
# Using gh CLI
gh pr create --repo mattclay/aws-terminator \
  --base main \
  --head <your-username>:add-<service>-terminators \
  --title "Add <service> terminators and permissions" \
  --body-file pr-description.md

# Or use GitHub web UI and paste pr-description.md content
```

#### 5. Link to Ansible Collection PR

Comment on ansible-collections/<collection>#<PR_NUMBER>:

```markdown
AWS Terminator PR: mattclay/aws-terminator#<NEW_PR_NUMBER>
```

**Tip**: Run with `--auto-pr` flag next time to automate commit/push/PR steps.
````

**With `--auto-pr` flag**: Automate git operations

Continue to automated steps:

#### Step 7a: Commit Changes

```bash
cd ~/dev/aws-terminator

git add aws/terminator/<modified-files> aws/policy/<modified-files>

git commit -m "Add <service> terminators and permissions

- Add <ResourceType>Terminator class to <terminator-file>.py
- Add IAM permissions for <service> operations to <policy-file>.yaml

Required for ansible-collections/<collection> PR #<PR_NUMBER>

Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
```

#### Step 7b: Push to Fork

```bash
git push origin "$BRANCH_NAME"
```

Output: `To github.com:<username>/aws-terminator.git`

#### Step 7c: Create Pull Request

```bash
gh pr create --repo mattclay/aws-terminator \
  --base main \
  --head <your-username>:$BRANCH_NAME \
  --title "Add <service> terminators and permissions" \
  --body-file pr-description.md
```

Capture PR number:

```bash
PR_URL=$(gh pr create ... | tail -1)
TERMINATOR_PR=$(echo "$PR_URL" | grep -o '[0-9]*$')
```

#### Step 7d: Link Back to Ansible Collection PR

```bash
gh pr comment <ANSIBLE_PR_NUMBER> --repo <ANSIBLE_REPO> --body "
## AWS Terminator PR Created

🔗 mattclay/aws-terminator#${TERMINATOR_PR}

### Added

**Terminator Classes**: <List>
**IAM Permissions**: <List>

### Status

- ✅ Tox validation passed
- ⏳ Awaiting review

Once merged, CI here should pass.

---
*Generated by /aws-terminator-workflow --auto-pr*
"
```

#### Step 7e: Workflow Summary

````markdown
## AWS Terminator Workflow Complete 🎉

**Ansible Collection PR**: ansible-collections/<collection>#<PR_NUMBER>
**aws-terminator PR**: mattclay/aws-terminator#<TERMINATOR_PR>
**Branch**: add-<service>-terminators

### What Was Done

1. ✅ Analyzed ansible collection PR
2. ✅ Implemented <N> terminator classes
3. ✅ Added <M> IAM permission blocks
4. ✅ Validated with tox
5. ✅ Committed changes
6. ✅ Pushed to fork
7. ✅ Created PR
8. ✅ Commented on ansible collection PR

### Next Steps

1. Monitor mattclay/aws-terminator#<TERMINATOR_PR>
2. Wait for review and merge
3. CI in ansible collection PR should pass after merge

---
*Workflow executed by /aws-terminator-workflow --auto-pr*
````

### Deployment (After Merge)

The aws-terminator maintainers will handle deployment, but for reference:

```bash
# Deploy permissions to dev
make test_policy STAGE=dev

# Test with ansible-test
ansible-test integration <module> --remote-stage dev

# Deploy to prod
make test_policy STAGE=prod

# Deploy lambda (if needed)
make terminator_lambda
```

## Configuration

Optional environment variables:

```bash
# Fork username (defaults to gh config)
export AWS_TERMINATOR_FORK="your-username"

# Local aws-terminator path (defaults to ~/dev/aws-terminator)
export AWS_TERMINATOR_PATH="~/custom/path"

# Enable auto-PR mode (default: false, requires manual commit/push/PR)
export AWS_TERMINATOR_AUTO_PR="true"
```

**Fork Setup**:

Before using this workflow, you must:

1. Fork mattclay/aws-terminator on GitHub to your account
2. The skills will automatically clone from YOUR fork (detected via `gh api user`)
3. The upstream remote is set to mattclay/aws-terminator for syncing
4. Changes are pushed to origin (your fork)
5. PR is created from your-fork:branch → mattclay/aws-terminator:main

## Flags

| Flag | Description |
| ---- | ----------- |
| `--pr <number>` | **Required** - Ansible collection PR number |
| `--repo <owner/repo>` | Repository (default: ansible-collections/community.aws) |
| `--auto-pr` | Automate commit, push, and PR creation (default: manual) |
| `--skip-tests` | Skip tox validation (faster, for drafts) |
| `--check` | Analysis only, don't implement |
| `--interactive` | Prompt for each implementation decision |

## Error Handling

### Analysis Failures

**No modules found**:

```
Analysis complete: No new modules found in PR
No aws-terminator changes needed.
```

**Action**: Exit gracefully, no work to do.

**Analysis errors**:

```
Error during analysis: <error message>
```

**Action**: Display error, offer to retry or abort.

### Implementation Failures

**Syntax errors**:

```
Python syntax error in aws/terminator/application_services.py:
  File "...", line 123
    def terminate(self)
                      ^
SyntaxError: invalid syntax
```

**Action**: Show error, offer to fix and retry.

**Tox failures**:

```
pylint failed:
  E1101: Module 'client' has no 'delete_foo' member
```

**Action**: Show error, offer options (fix/skip/abort).

### Git/PR Failures

**Push failures**:

```
Error: failed to push branch
Permission denied (publickey)
```

**Action**: Show error, suggest checking gh auth status.

**PR creation failures**:

```
Error creating PR: GraphQL error
```

**Action**: Show error, provide manual PR creation instructions.

## Recovery

### Resume from Failure

If workflow fails mid-execution, it can be resumed:

```bash
# Check current state
cd ~/dev/aws-terminator
git status

# Resume workflow from where it left off
/aws-terminator-workflow --pr <PR_NUMBER> --resume
```

State is tracked in `.aws-terminator-workflow-state.json`:

```json
{
  "pr_number": 2353,
  "repo": "ansible-collections/community.aws",
  "last_completed_step": "implement",
  "branch_name": "add-medialive-terminators",
  "analysis_file": "/tmp/terminator-analysis-2353.md"
}
```

## Related Skills

- `/aws-terminator-analyze` - Just analyze (no implementation)
- `/aws-terminator-implement` - Just implement (analysis already done)

## References

- aws-terminator repository: https://github.com/mattclay/aws-terminator
- Ansible Cloud Content Handbook: https://github.com/ansible-collections/cloud-content-handbook
- Ansible Collections: amazon.aws, community.aws, amazon.ai, amazon.cloud
