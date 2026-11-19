# GitHub Fork Syncer

A lightweight, automated GitHub Action workflow that keeps all of your non-archived GitHub repository forks up to date with their upstream sources.

## How It Works

Instead of adding workflow files to every forked repository, this project acts as a central control repository. On a daily schedule (or when triggered manually), it uses the GitHub CLI (`gh`) to fetch a list of all active forks owned by your account and syncs their default branches with upstream.

## Setup Instructions

### 1. Generate a Personal Access Token (PAT)
1. Open your GitHub **Developer settings** > **Personal access tokens** > **Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Give the token a descriptive name (e.g., `Fork Syncer`) and select the following scopes:
   - `repo` (required to sync private and public forks)
   - `workflow` (required if upstream repositories contain updates inside `.github/workflows/`)
4. Copy the generated token.

### 2. Add the Repository Secret
1. Go to this repository's main page on GitHub and click the **Settings** tab under the repository name.
2. In the left sidebar under **Security**, click **Secrets and variables** > **Actions**.
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
          # List all non-archived forks owned by the authenticated user
          gh repo list --fork --no-archived --limit 1000 --json nameWithOwner -q '.[].nameWithOwner' | while read -r repo; do
            echo "Syncing $repo..."
            gh repo sync "$repo" || echo "Skipped $repo (likely has merge conflicts or custom changes)"
          done
```

## Usage

- **Automatic Runs:** The workflow runs automatically every day at `00:00 UTC`.
- **Manual Runs:** Navigate to the **Actions** tab in this repository, select **Sync All Forks** on the left, and click **Run workflow**.

## Behavior & Caveats

- **Merge Conflicts:** Forks whose default branches have diverged with conflicting commits are safely skipped without failing the rest of the batch.
- **Force Overwriting (Optional):** If you want your forks to strictly mirror upstream and discard custom commits on default branches, append `--force` to the sync command (`gh repo sync "$repo" --force`).
- **Archived Repositories:** Archived forks are automatically ignored (`--no-archived`).
