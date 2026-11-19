# Fork Syncer

Automatically keep all of your GitHub forks up to date with their upstream repositories using a single central GitHub Actions workflow.

## Overview

Instead of adding workflow files to every forked repository, this project runs a scheduled GitHub Action from one central control repository. It queries all active (non-archived) forks owned by your account and calls GitHub's native `merge-upstream` REST API on each repository's default branch.

### Key Features

* **Centralized automation:** Manages every fork across your account from a single workflow file.
* **Clean commit history:** Uses GitHub's native upstream merge API rather than pushing extra sync commits.
* **Detailed logging & summary table:** Distinguishes between forks that were **Updated**, **Already up to date**, or **Skipped** due to merge conflicts, outputting results to both the console logs and a formatted **GitHub Actions Job Summary** table.
* **Safe conflict handling:** Automatically skips forks where your default branch has diverged with conflicting commits.

---

## Setup Instructions

### 1. Create a Personal Access Token (PAT)

1. Open your GitHub **Settings** (from your profile menu in the top-right corner).
2. Scroll down to **Developer settings** in the left sidebar.
3. Navigate to **Personal access tokens > Tokens (classic)**.
4. Click **Generate new token (classic)** and select the following scopes:
   * `repo` (required to access and update private and public forks)
   * `workflow` (required if any upstream repositories contain changes under `.github/workflows/`)
5. Generate and copy the token.

### 2. Add the Repository Secret

1. Open the **Settings** tab inside this control repository (not your personal account settings).
2. In the left sidebar under **Security**, click **Secrets and variables > Actions**.
3. Click **New repository secret**.
4. Set the **Name** to `SYNC_TOKEN` and paste your Personal Access Token into the **Secret** field.
5. Click **Add secret**.

### 3. Add the Workflow File

Ensure `.github/workflows/sync-forks.yml` exists in this repository with the following configuration:

```yaml
name: Sync All Forks

on:
  schedule:
    # Runs every day at 00:00 UTC
    - cron: '0 0 * * *'
  workflow_dispatch: # Allows manual runs from the Actions tab

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Fetch and sync all forks
        env:
          GH_TOKEN: ${{ secrets.SYNC_TOKEN }}
        run: |
          echo "## 🔄 Fork Sync Summary" >> $GITHUB_STEP_SUMMARY
          echo "| Repository | Branch | Status |" >> $GITHUB_STEP_SUMMARY
          echo "| :--- | :--- | :--- |" >> $GITHUB_STEP_SUMMARY

          gh repo list --fork --no-archived --limit 1000 --json nameWithOwner,defaultBranchRef \
            -q '.[] | "\(.nameWithOwner) \(.defaultBranchRef.name)"' | while read -r repo branch; do
            
            # Call the merge-upstream endpoint directly to get the exact status message
            response=$(gh api -X POST "repos/$repo/merge-upstream" -f branch="$branch" 2>&1)
            exit_code=$?

            if [ $exit_code -eq 0 ]; then
              msg=$(echo "$response" | jq -r '.message')
              if [[ "$msg" == *"Already up to date"* ]]; then
                echo "⏭️ $repo ($branch): Already up to date"
                echo "| \`$repo\` | \`$branch\` | ⏭️ Up to date |" >> $GITHUB_STEP_SUMMARY
              else
                echo "✅ $repo ($branch): Updated! ($msg)"
                echo "| \`$repo\` | \`$branch\` | ✅ **Updated** |" >> $GITHUB_STEP_SUMMARY
              fi
            else
              err=$(echo "$response" | jq -r '.message // empty' 2>/dev/null || echo "$response")
              echo "⚠️ $repo ($branch): Failed - $err"
              echo "| \`$repo\` | \`$branch\` | ⚠️ Skipped (\`$err\`) |" >> $GITHUB_STEP_SUMMARY
            fi
          done
```

---

## Usage & Monitoring

* **Scheduled Runs:** The workflow runs automatically every day at `00:00 UTC`. Edit the `cron` expression in `.github/workflows/sync-forks.yml` to change the schedule.
* **Manual Runs:** Go to the **Actions** tab in this repository, select **Sync All Forks**, and click **Run workflow**.
* **Viewing Sync Results:** Click on any completed workflow run in the **Actions** tab to inspect the **Fork Sync Summary** table at the bottom of the run summary page:
  * ✅ **Updated:** New upstream commits were fast-forwarded or merged into your fork.
  * ⏭️ **Up to date:** Your fork already matched the upstream branch.
  * ⚠️ **Skipped:** The sync could not complete automatically (usually due to merge conflicts or custom commits on the default branch).

---

## Troubleshooting & Notes

* **Archived repositories:** Archived forks are automatically ignored (`--no-archived`).
* **Merge conflicts:** If a fork shows `⚠️ Skipped`, open that fork on GitHub and resolve the conflict manually or discard your local changes via **Sync fork > Discard commits**.
