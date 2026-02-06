# Local Testing Setup: Automerge Retry Logic

Step-by-step instructions to test the **Validate and Automerge** workflow, including retry-on-conflict behavior. Uses only Git and the GitHub web interface (no `gh` CLI on your machine).

---

## Prerequisites

- Git installed
- GitHub account (free tier is fine)
- No Python/Node/GitHub CLI required

---

## Part 1: Create the repository on GitHub

1. **Go to GitHub:** [https://github.com/new](https://github.com/new)

2. **Create a new repository:**
   - **Repository name:** `test-automerge-retry`
   - **Description:** (optional) "Test automerge workflow with retry"
   - **Visibility:** Public (or Private; both work)
   - **Do not** add a README, .gitignore, or license (we’ll push existing content)
   - Click **Create repository**

3. **Note the repo URL** (e.g. `https://github.com/YOUR_USERNAME/test-automerge-retry.git`). You’ll use it in Part 2.

---

## Part 2: Set up files locally with Git

Run these in a terminal. Replace `YOUR_USERNAME` and the URL with your own.

```bash
# Go to the folder that contains the project (e.g. where you cloned or created it)
cd /path/to/test-automerge-retry

# Initialize Git (if not already a repo)
git init

# Create/ensure branch name is main
git branch -M main

# Add all files
git add .
git commit -m "Initial commit: workflow and test spec"

# Add your GitHub repo as remote (replace YOUR_USERNAME with your GitHub username)
git remote add origin https://github.com/YOUR_USERNAME/test-automerge-retry.git

# Push (GitHub may show you a command to set upstream; use it if needed)
git push -u origin main
```

If the folder already has the files from this setup, the only thing you might need is:

```bash
cd /path/to/test-automerge-retry
git init
git branch -M main
git add .
git commit -m "Initial commit: workflow and test spec"
git remote add origin https://github.com/YOUR_USERNAME/test-automerge-retry.git
git push -u origin main
```

---

## Part 3: Configure repository settings (GitHub web)

1. **Open the repo on GitHub:**  
   `https://github.com/YOUR_USERNAME/test-automerge-retry`

2. **Settings → General → Pull Requests**
   - Under “Allow merge commits”, ensure **Allow merge commits** is checked.
   - (Optional) Uncheck “Allow squash merging” / “Allow rebase merging” if you want only merge commits for this test.

3. **Permissions for GITHUB_TOKEN**
   - Go to **Settings → Actions → General**.
   - Under “Workflow permissions”, select **Read and write permissions**.
   - Click **Save**.

This allows the workflow’s `GITHUB_TOKEN` to merge PRs and update branches.

---

## Part 4: Commands to create the file structure (reference)

If you prefer to create the repo from scratch with shell commands:

```bash
# Create directory structure
mkdir -p test-automerge-retry/.github/workflows
mkdir -p test-automerge-retry/specs/test-app/latest

# Create workflow (paste the YAML content into the file, or use the file from this repo)
# Create api.yaml
cat > test-automerge-retry/specs/test-app/latest/api.yaml << 'EOF'
openapi: 3.0.0
info:
  title: Test App API
  version: 1.0.0
paths: {}
EOF

# Create README
echo "# test-automerge-retry" > test-automerge-retry/README.md
echo "See SETUP.md for testing instructions." >> test-automerge-retry/README.md

cd test-automerge-retry
git init
git branch -M main
git add .
git commit -m "Initial commit"
# Then add remote and push as in Part 2
```

The workflow file content is in `.github/workflows/validate-api-specs.yaml` in this repo; copy it into the same path in `test-automerge-retry`.

---

## Part 5: Create two PRs (first merges cleanly, second triggers retry)

Goal: **PR #1** merges cleanly. **PR #2** is based on an older `main`, so the first merge attempt fails (branch out of date / not mergeable), then the workflow updates the branch and retries.

### 5.1 First PR (will merge cleanly)

```bash
cd /path/to/test-automerge-retry   # same repo as above, on main, up to date

# Create branch for first PR
git checkout -b sync-pr-first
```

Edit `specs/test-app/latest/api.yaml` and add a description under `info:` (e.g. after `version: 1.0.0`):

```yaml
openapi: 3.0.0
info:
  title: Test App API
  version: 1.0.0
  description: First sync
paths: {}
```

Then:

```bash
git add specs/test-app/latest/api.yaml
git commit -m "Sync API specs: first change"
git push -u origin sync-pr-first
```

**On GitHub:**

1. You should see a prompt to **Compare & pull request** for `sync-pr-first`.
2. Open the PR.
3. **Title must start with:** `Sync API specs for `  
   Example: **`Sync API specs for test-app`**
4. Base: `main`, compare: `sync-pr-first`.
5. Create the pull request.

**Expected:** The “Validate and Automerge” workflow runs, validation passes, and the automerge job merges the PR. PR #1 is merged into `main`.

---

### 5.2 Second PR (will be “out of date” and trigger retry)

This PR must be created **from the same `main` that existed before PR #1 was merged**, so that after PR #1 is merged, PR #2’s base is behind `main` and the first merge attempt fails.

**Option A – Create second branch before merging PR #1 (recommended)**

Before (or right after) opening PR #1, create the second branch from current `main` and push it:

```bash
# From your repo root, on main (before or after opening PR #1, but before PR #1 is merged)
git checkout main
git pull origin main   # ensure you're on latest main

# Create second branch from current main
git checkout -b sync-pr-second
```

Edit `specs/test-app/latest/api.yaml` with a **different** change (so it will diverge from what PR #1 will add):

```yaml
openapi: 3.0.0
info:
  title: Test App API
  version: 1.0.0
  description: Second sync
paths: {}
```

Then:

```bash
git add specs/test-app/latest/api.yaml
git commit -m "Sync API specs: second change"
git push -u origin sync-pr-second
```

**Do not open the second PR yet.** Wait until PR #1 has been merged by the workflow.

After **PR #1 is merged**:

1. On GitHub, create a new PR: base **`main`**, compare **`sync-pr-second`**.
2. **Title:** e.g. **`Sync API specs for test-app v2`** (must start with `Sync API specs for `).
3. Create the pull request.

Now `main` has “First sync” and `sync-pr-second` has “Second sync” in the same place. The PR is behind `main` and may show as “out of date” or having merge conflicts. The workflow will try to merge, then (when it hits the failure) run the retry logic.

**Option B – Create second branch after PR #1 is merged**

If you already merged PR #1:

```bash
git checkout main
git pull origin main
git checkout -b sync-pr-second
# Make the "Second sync" change to api.yaml as above
git add specs/test-app/latest/api.yaml
git commit -m "Sync API specs: second change"
git push -u origin sync-pr-second
```

Then open the PR on GitHub with title starting with `Sync API specs for `. To really test “out of date” and retry, make a small extra edit on `main` to `specs/test-app/latest/api.yaml` and push, so that `sync-pr-second` is behind `main` when the workflow runs.

---

## Part 6: Observe the workflow in GitHub Actions

1. **Open the repo on GitHub** → **Actions** tab.

2. **Workflow runs**
   - Each run is listed as “Validate and Automerge”.
   - Click a run to see jobs: **validate** and **automerge**.

3. **For PR #1 (clean merge)**
   - **validate:** “Validating specs…” and “Validation passed”.
   - **automerge:** “Attempting to merge PR #1…” then “PR merged successfully”. No retry.

4. **For PR #2 (retry path)**
   - **validate:** Same as above.
   - **automerge:** You should see something like:
     - “Attempting to merge PR #2…”
     - “First merge attempt failed: …” (message may mention “out of date”, “not mergeable”, or “merge conflict”).
     - “Merge conflict detected, updating branch…”
     - “Branch updated, waiting for GitHub to process…”
     - “Retrying merge…”
     - Either “PR merged successfully after branch update” or “Retry failed: …”.

---

## Part 7: What to look for in the logs

### Validation job

- “Validating specs…”
- “:white_check_mark: Validation passed” (or “Error: api.yaml not found” if the file is missing).

### Automerge job – first attempt

- “Attempting to merge PR #&lt;n&gt;…”
- Either “PR merged successfully” or “First merge attempt failed: &lt;output&gt;”.

### Automerge job – retry (only when first merge fails)

- “:warning: Merge conflict detected, updating branch…”
- “Branch updated, waiting for GitHub to process…”
- “Retrying merge…”
- Then either “PR merged successfully after branch update” or “:x: Retry failed: …”.

If the merge never succeeds you may see “:x: PR merge failed” and exit code 1.

---

## Part 8: How to verify the retry logic worked

1. **Logs:** In the **automerge** step log you see:
   - “First merge attempt failed”,
   - then “Merge conflict detected, updating branch…”,
   - then “Branch updated…”,
   - then “Retrying merge…”,
   - then “PR merged successfully after branch update”.

2. **PR state:** The second PR changes from “Merge conflict” or “Out of date” to **Merged** (after the workflow run).

3. **Branches:** After merge, `main` contains the changes from both PRs (and the branch update/merge done by the workflow).

4. **Commit history:** On `main`, you should see merge commits (or merge commit + update) for the second PR, consistent with the workflow having updated the branch and then merged.

---

## Quick reference: workflow triggers

- **When:** `pull_request` with `opened`, `synchronize`, or `reopened`.
- **Path filter:** Only when files under `specs/**` change.
- **Title filter:** Only when the PR title starts with **`Sync API specs for `** (including the space).

If a run doesn’t appear, check that the PR title and changed files match these conditions.

---

## Troubleshooting

| Issue | What to check |
|-------|----------------|
| Workflow doesn’t run | PR title must start with “Sync API specs for ”; PR must touch files under `specs/**`. |
| “Permission denied” / merge fails | Repo **Settings → Actions → General** → Workflow permissions set to **Read and write**. |
| Merge method not allowed | **Settings → General → Pull Requests** → allow **merge commits**. |
| Retry path never runs | Second PR must be behind `main` (create branch before merging first PR, or make an extra commit on `main` so the second branch is out of date). |

---

All commands are copy-paste ready; only replace `YOUR_USERNAME` and paths with your values. Use the GitHub web UI for creating the repo, opening PRs, and changing settings.
