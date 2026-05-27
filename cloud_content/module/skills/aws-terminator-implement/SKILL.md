---
name: aws-terminator-implement
description: Implement terminator classes and IAM permissions in aws-terminator repository based on analysis
allowed-tools: Read, Edit, Write, Bash(command:git *), Bash(command:gh *), Bash(command:grep *)
argument-hint: "[--analysis <file>] [--interactive]"
triggers:
  - "implement terminator"
  - "aws-terminator implement"
  - "create terminator classes"
  - "add terminator permissions"
---

# AWS Terminator Implement

Implements terminator classes and IAM permissions in the mattclay/aws-terminator repository based on analysis from `/aws-terminator-analyze` or manual specification.

## Purpose

After analyzing an Ansible collection PR with `/aws-terminator-analyze`, this skill:

1. Creates terminator class implementations in the appropriate file
2. Adds IAM permissions to the appropriate policy file
3. Follows aws-terminator patterns and conventions
4. Creates properly formatted code ready for PR submission

## Quick Start

```bash
# Implement based on prior analysis (from /aws-terminator-analyze)
/aws-terminator-implement

# Interactive mode (prompts for each decision)
/aws-terminator-implement --interactive
```

**What it does**: Creates terminator class implementations, adds IAM permissions to policy files, runs validation tests, and prepares code for PR submission.

**Prerequisites**: Fork mattclay/aws-terminator, clone your fork, and run `/aws-terminator-analyze` first.

**See full documentation below** for terminator class patterns, IAM permission structure, and validation steps.

## When to Use

- After running `/aws-terminator-analyze` and reviewing the report
- When you know what terminators and permissions are needed
- To implement recommendations from analysis
- Before submitting a PR to mattclay/aws-terminator

## Usage

```bash
# Implement based on prior analysis
/aws-terminator-implement

# Implement with saved analysis file
/aws-terminator-implement --analysis terminator-analysis.md

# Interactive mode (prompt for each implementation)
/aws-terminator-implement --interactive
```

## Prerequisites

- **Fork** of mattclay/aws-terminator repository (fork it on GitHub first)
- aws-terminator fork cloned at `~/dev/aws-terminator`
- Analysis completed (from `/aws-terminator-analyze` or manual)
- Git configured with user credentials
- `gh` CLI authenticated (for PR creation)

## Step 1: Setup and Validation

**Check aws-terminator repository**:

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
git fetch origin
git checkout main
git pull origin main
```

**Create implementation branch**:

```bash
# Branch name format: add-<service>-terminators
# Example: add-medialive-terminators
git checkout -b add-<SERVICE>-terminators
```

## Step 2: Load Analysis Context

If `--analysis <file>` provided, read the analysis file.

Otherwise, use context from previous `/aws-terminator-analyze` invocation in this conversation.

**Required information**:

- Resource types to add terminators for
- AWS service boto3 client name
- Resource properties (id, name, created_time fields)
- Terminator file to modify (compute.py, application_services.py, etc.)
- IAM permissions to add
- Policy file to modify (compute.yaml, application-services.yaml, etc.)

**If missing context**, prompt user for:

```
I need the following information to implement terminators:

1. AWS Service: (e.g., medialive, rds, ec2)
2. Resource Types: (e.g., Cluster, Input, Network)
3. Terminator File: (e.g., application_services.py)
4. Policy File: (e.g., application-services.yaml)

You can provide this by:
- Running /aws-terminator-analyze first
- Providing an analysis file with --analysis
- Entering the details manually
```

## Step 3: Implement Terminator Classes

For each resource type, generate and add the terminator class.

### Determine Base Class

**Read resource describe output** to check for timestamp field:

```bash
# If analysis includes example API response, check for:
# CreatedTime, LaunchTime, StartTime, CreationDate, CreatedAt, etc.
```

**Decision**:

- Has timestamp → use `Terminator` base class
- No timestamp → use `DbTerminator` base class

### Generate Terminator Class Code

**Template for `Terminator` base class**:

```python
class <ResourceType>Terminator(Terminator):
    @staticmethod
    def create(credentials):
        def get_<resource_type_snake_case>(client):
            # Use paginator if API supports it and can return >default limit
            # Otherwise use simple list call
            <list_call> = client.<list_operation>()<'ResponseKey'>
            return <list_call>
        
        return Terminator._create(
            credentials,
            <ResourceType>Terminator,
            '<boto3-service-name>',
            get_<resource_type_snake_case>
        )
    
    @property
    def id(self):
        return self.instance['<IdField>']
    
    @property
    def name(self):
        # Prefer Name field, fallback to id
        return self.instance.get('<NameField>', self.id)
    
    @property
    def created_time(self):
        return self.instance['<CreatedTimeField>']
    
    def terminate(self):
        # Add any pre-delete requirements (e.g., stop, detach)
        # Then delete
        self.client.<delete_operation>(<IdParam>=self.id)
```

**Template for `DbTerminator` base class**:

```python
class <ResourceType>Terminator(DbTerminator):
    @staticmethod
    def create(credentials):
        def get_<resource_type_snake_case>(client):
            <list_call> = client.<list_operation>()<'ResponseKey'>
            return <list_call>
        
        return Terminator._create(
            credentials,
            <ResourceType>Terminator,
            '<boto3-service-name>',
            get_<resource_type_snake_case>
        )
    
    @property
    def name(self):
        return self.instance['<NameField>']
    
    def terminate(self):
        self.client.<delete_operation>(<NameParam>=self.name)
```

### Add Class to Terminator File

**Locate insertion point**:

```bash
cd ~/dev/aws-terminator
# Read the appropriate terminator file
# Classes are typically alphabetically ordered
# Insert new class in alphabetical position
```

**Use Edit tool** to add the class:

- Read the terminator file
- Find insertion point (alphabetically by class name)
- Insert the new class code

### Handle Special Cases

**Pagination required**:

```python
def get_resources(client):
    paginator = client.get_paginator('list_resources')
    resources = paginator.paginate(MaxResults=100).build_full_result()['Resources']
    return resources
```

**Filter by account owner**:

```python
from aws.terminator.util import get_account_id

def get_resources(client):
    account = get_account_id(credentials)
    return client.describe_resources(OwnerIds=[account])['Resources']
```

**Flattening nested lists**:

```python
def get_resources(client):
    return [item for group in client.describe_groups()['Groups'] 
            for item in group['Items']]
```

**Pre-delete operations**:

```python
def terminate(self):
    # Example: Stop resource before deleting
    if self.instance['State'] != 'stopped':
        self.client.stop_resource(ResourceId=self.id)
        # Wait for stopped state if needed
    
    self.client.delete_resource(ResourceId=self.id)
```

**Ignore terminated/deleted resources**:

```python
@property
def ignore(self):
    return self.instance['State'] in ['terminated', 'deleted']
```

**Custom age limit**:

```python
@property
def age_limit(self):
    # Default is 20 minutes
    # Override for resources that need longer to provision
    return datetime.timedelta(minutes=50)
```

## Step 4: Implement IAM Permissions

### Determine Policy Structure

**Permissions are grouped into blocks**:

1. **Resource-scoped permissions** - Actions on specific ARNs
2. **Global permissions** - List/Describe actions that require `Resource: "*"`

### Generate Permission Blocks

**Resource-scoped permission block**:

```yaml
- Sid: <ServiceName><ResourceType>Permissions
  Effect: Allow
  Action:
    - <service>:Create<Resource>
    - <service>:Delete<Resource>
    - <service>:Describe<Resource>
    - <service>:Update<Resource>
    - <service>:Tag Resource
    - <service>:UntagResource
  Resource:
    - 'arn:aws:<service>:{{ aws_region }}:{{ aws_account_id }}:<resource-type>/*'
```

**Global permission block**:

```yaml
- Sid: <ServiceName><ResourceType>GlobalPermissions
  Effect: Allow
  Action:
    - <service>:List<Resources>
    - <service>:Describe<Resource>  # If doesn't require resource ARN
  Resource:
    - '*'
```

### Add Permissions to Policy File

**Locate policy file**:

```bash
cd ~/dev/aws-terminator
# Read aws/policy/<appropriate-file>.yaml
```

**Use Edit tool** to add permissions:

- Find appropriate insertion point (alphabetically by Sid)
- Insert new permission blocks
- Ensure no duplicate permissions (check existing blocks first)

### Permission Best Practices

**Least privilege**:

- Only add actions actually used by tests or terminators
- Avoid wildcards like `Delete*` unless necessary
- Scope resources to specific ARN patterns

**Action naming patterns**:

```yaml
# Prefer explicit actions
- service:CreateFoo
- service:DeleteFoo
- service:DescribeFoo

# Over wildcards
- service:*  # Too broad
```

**Resource ARN patterns**:

```yaml
# Specific resource type
Resource:
  - 'arn:aws:service:{{ aws_region }}:{{ aws_account_id }}:foo/*'

# Multiple resource types
Resource:
  - 'arn:aws:service:{{ aws_region }}:{{ aws_account_id }}:foo/*'
  - 'arn:aws:service:{{ aws_region }}:{{ aws_account_id }}:bar/*'

# Global (only for List/Describe)
Resource:
  - '*'
```

## Step 5: Update Imports (if needed)

**Check if new imports are needed**:

```bash
cd ~/dev/aws-terminator
grep -n "^from " aws/terminator/<terminator-file>.py | head -10
grep -n "^import " aws/terminator/<terminator-file>.py | head -10
```

**Common imports**:

```python
import datetime
from aws.terminator import Terminator, DbTerminator
from aws.terminator.util import get_account_id
```

**Add imports if missing** (use Edit tool at top of file).

## Step 6: Validation

### Syntax Check

**Python syntax**:

```bash
cd ~/dev/aws-terminator
python3 -m py_compile aws/terminator/<terminator-file>.py
```

**YAML syntax**:

```bash
cd ~/dev/aws-terminator
python3 -c "import yaml; yaml.safe_load(open('aws/policy/<policy-file>.yaml'))"
```

### Run Tox Tests

```bash
cd ~/dev/aws-terminator
tox
```

Expected checks:

- pycodestyle - Python style
- pylint - Code quality
- yamllint - YAML style
- policy - Policy validation

### Check for Duplicates

**Duplicate terminators**:

```bash
grep -r "class <ResourceType>" aws/terminator/
# Should only find the new one
```

**Duplicate permissions**:

```bash
grep -r "<service>:<action>" aws/policy/
# Check if permission already exists elsewhere
```

## Step 7: Generate Implementation Summary

Create a summary of what was implemented:

````markdown
## AWS Terminator Implementation Summary

**Branch**: add-<service>-terminators
**Files Modified**:
- `aws/terminator/<terminator-file>.py`
- `aws/policy/<policy-file>.yaml`

### Terminator Classes Added

#### 1. <ResourceType>Terminator

**File**: `aws/terminator/<terminator-file>.py`
**Base Class**: `Terminator` | `DbTerminator`
**Lines Added**: ~30

**Properties**:
- `id`: Returns `<IdField>` from instance
- `name`: Returns `<NameField>` from instance
- `created_time`: Returns `<TimestampField>` from instance (if Terminator)

**Termination Logic**:
```python
def terminate(self):
    self.client.<delete_operation>(<IdParam>=self.id)
```

**Special Handling**:
- [None | Pagination | Pre-delete stop | Ignore terminated]

#### 2. (Additional classes if multiple)

### IAM Permissions Added

**File**: `aws/policy/<policy-file>.yaml`

**Permissions Block 1** - Resource-scoped:
```yaml
Sid: <ServiceName><ResourceType>Permissions
Actions: <N> actions (<service>:Create*, Delete*, Describe*, Update*, Tag*)
Resources: arn:aws:<service>:region:account:<resource-type>/*
```

**Permissions Block 2** - Global (if applicable):
```yaml
Sid: <ServiceName>GlobalPermissions
Actions: <N> actions (<service>:List*, Describe*)
Resources: *
```

### Testing Checklist

**Before submitting PR**:

- [ ] Python syntax valid (`python3 -m py_compile`)
- [ ] YAML syntax valid
- [ ] Tox tests pass (`tox`)
- [ ] No duplicate terminator classes
- [ ] No duplicate permission blocks
- [ ] Imports added (if needed)
- [ ] Classes in alphabetical order
- [ ] Permissions in alphabetical order by Sid

**Manual testing** (if possible):
```bash
# Create test resources in AWS account
# Then run terminator in check mode
cd ~/dev/aws-terminator/aws
python cleanup.py --stage dev --target <ResourceType>Terminator -v -c

# Should output:
# cleanup: DEBUG located <ResourceType>Terminator: count=X
# cleanup: DEBUG checked <ResourceType>Terminator: name=..., id=..., age=..., stale=True/False
```

### Next Steps

1. **Review changes**:
   ```bash
   cd ~/dev/aws-terminator
   git diff
   ```

2. **Commit changes**:
   ```bash
   git add aws/terminator/<terminator-file>.py
   git add aws/policy/<policy-file>.yaml
   git commit -m "Add <service> terminators and permissions
   
   - Add <ResourceType>Terminator class
   - Add IAM permissions for <service> operations
   - Required for ansible-collections/<collection> PR #<NUMBER>
   
   Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"
   ```

3. **Push branch**:
   ```bash
   git push origin add-<service>-terminators
   ```

4. **Create PR**:
   ```bash
   gh pr create --repo mattclay/aws-terminator \
     --title "Add <service> terminators and permissions" \
     --body "..."
   ```

---
*Implementation generated by /aws-terminator-implement skill*
````

## Interactive Mode

If `--interactive` flag is provided, prompt before each change:

```
Ready to implement <ResourceType>Terminator in <terminator-file>.py

Properties:
- id: <IdField>
- name: <NameField>
- created_time: <TimestampField>

Terminate operation: client.<delete_operation>(<IdParam>=self.id)

Proceed with implementation? [Y/n]:
```

## Configuration

Optional environment variables:

```bash
# Local aws-terminator path (defaults to ~/dev/aws-terminator)
export AWS_TERMINATOR_PATH="~/custom/path"
```

If not set, the skill uses `~/dev/aws-terminator` as the default location.

**Fork Setup**:

This skill requires a fork of mattclay/aws-terminator:

1. Fork mattclay/aws-terminator on GitHub to your account
2. Clone from YOUR fork (detected via `gh api user`)
3. Upstream remote set to mattclay/aws-terminator
4. Push changes to origin (your fork)
5. Create PR from your fork to mattclay/aws-terminator

## Error Handling

**aws-terminator not found**:

```
Error: aws-terminator repository not found at ~/dev/aws-terminator

Clone it with:
  git clone https://github.com/mattclay/aws-terminator.git ~/dev/aws-terminator

Then re-run this skill.
```

**Dirty working tree**:

```
Error: aws-terminator repository has uncommitted changes

Commit or stash changes first:
  cd ~/dev/aws-terminator
  git status
  git stash  # or git commit
```

**Missing analysis context**:

```
Error: No analysis context found

Run /aws-terminator-analyze first, or provide:
  --analysis <file> pointing to analysis output
```

## Related Skills

- `/aws-terminator-analyze` - Analyze Ansible collection PR to determine what's needed
- `/aws-terminator-workflow` - Orchestrator that runs analyze + implement + PR creation

## References

- aws-terminator repository: https://github.com/mattclay/aws-terminator
- Terminator base classes: `aws/terminator/__init__.py`
- Existing terminators: `aws/terminator/*.py`
- Policy structure: `aws/policy/*.yaml`
