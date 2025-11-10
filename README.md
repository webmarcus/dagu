# Dagu DAG Repository

This repository contains DAG (Directed Acyclic Graph) workflow definitions for Dagu scheduler.

## Overview

DAGs in this repository are automatically synchronized to the Dagu coordinator using:
- **Initial Deploy**: Full sync triggered by Ansible on first deployment
- **Incremental Updates**: GitHub Actions syncs only changed files on every commit
- **Optional Backup**: Scheduled full sync (if configured)

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       └── sync-to-dagu.yml    # GitHub Actions workflow for incremental sync
├── test-dag.yaml               # Example: Basic DAG features
├── distributed-test.yaml       # Example: Distributed execution with worker labels
└── README.md                   # This file
```

## Test DAGs

### test-dag.yaml
Basic workflow demonstrating:
- Environment variables
- Parameters
- Sequential steps
- Parallel execution
- Error handlers
- Dependencies

**Run manually:**
```bash
# From Dagu UI: http://coordinator:8080
# Or via API:
curl -X POST http://coordinator:8080/api/v2/dags/test-dag.yaml/start
```

### distributed-test.yaml
Distributed workflow demonstrating:
- Worker label routing
- Regional task placement
- CPU/GPU task distribution
- Graceful fallback handling

**Prerequisites:**
- Coordinator with gRPC enabled
- Workers with appropriate labels (region, type, gpu)

## Adding New DAGs

1. Create YAML file in repository root:
   ```yaml
   name: my-workflow
   description: "My custom workflow"

   steps:
     - name: step1
       command: echo "Hello World"
   ```

2. Commit and push:
   ```bash
   git add my-workflow.yaml
   git commit -m "Add my-workflow DAG"
   git push origin main
   ```

3. GitHub Actions automatically syncs to coordinator (within seconds)

## DAG Development

### Local Testing
```bash
# Validate DAG syntax
dagu validate my-workflow.yaml

# Test run locally
dagu start my-workflow.yaml
```

### Best Practices
- Use descriptive step names
- Add comments for complex logic
- Set appropriate `histRetentionDays`
- Use parameters for reusability
- Add error handlers
- Tag DAGs for organization

### Worker Labels
Route tasks to specific workers using `requiredLabels`:

```yaml
steps:
  - name: gpu-task
    command: python train.py
    requiredLabels:
      gpu: "nvidia-a100"
      vram: "80GB"
```

Common labels:
- `type`: general, cpu-intensive, gpu
- `region`: eu-north-1, us-east-1, etc.
- `gpu`: nvidia-a100, nvidia-t4, etc.
- `cores`: Number of CPU cores
- `ram`: Available RAM

## Sync Mechanisms

### Initial Deployment
When Ansible deploys coordinator for first time:
```bash
ansible-playbook dagu-coordinator.yml
```
All DAGs synced via: `sync-dags-from-git.yaml` DAG trigger

### Incremental Sync (GitHub Actions)
On every commit to `main`:
1. Detects changed YAML files
2. Finds coordinator via Tailscale API (tag:dagu-coordinator)
3. Transfers only changed/new/deleted files via SCP
4. Updates visible in Dagu UI within seconds

### Scheduled Backup (Optional)
If `git_sync_schedule` is configured (e.g., `"0 3 * * *"`):
- Periodic full clone from Git repository
- Safety net for missed updates
- Configurable in Ansible group_vars

## GitHub Actions Setup

### 1. Create Tailscale OAuth Client
https://login.tailscale.com/admin/settings/oauth

Settings:
- Name: "GitHub Actions - DAG Sync"
- Scopes: `devices (read)`
- Tags: `tag:ci`

Copy the generated Client ID and Client Secret.

### 2. Store Credentials in Bitwarden Secrets Manager

Create two secrets in Bitwarden:
1. `TS_OAUTH_CLIENT_ID` - Paste the Tailscale OAuth client ID
2. `TS_OAUTH_SECRET` - Paste the Tailscale OAuth client secret

Note the Secret IDs (UUIDs) for each secret.

### 3. Configure GitHub Repository

**GitHub Secrets** (Settings → Secrets and variables → Actions):
- `BWS_ACCESS_TOKEN` - Your Bitwarden Secrets Manager access token
  - Get from: Bitwarden web vault → Machine accounts

**Update Workflow** (`.github/workflows/sync-to-dagu.yml`):
- Replace `YOUR-TS-OAUTH-CLIENT-ID-SECRET-ID` with actual Bitwarden secret ID
- Replace `YOUR-TS-OAUTH-SECRET-SECRET-ID` with actual Bitwarden secret ID

This approach centralizes all secrets in Bitwarden, consistent with Ansible configuration.

## Monitoring

### Dagu UI
```
http://coordinator:8080
```

### Check Sync Status
```bash
# View sync DAG history
curl http://coordinator:8080/api/v2/dags/sync-dags-from-git.yaml/history

# Manual trigger
curl -X POST http://coordinator:8080/api/v2/dags/sync-dags-from-git.yaml/start
```

### Logs
```bash
# Coordinator logs
ssh coordinator 'journalctl -u dagu -f'

# Worker logs
ssh worker 'journalctl -u dagu -f'
```

## Documentation

- **Dagu Docs**: https://dagu.readthedocs.io/
- **DAG Syntax**: https://dagu.readthedocs.io/en/latest/config.html
- **Distributed Execution**: https://dagu.readthedocs.io/en/latest/distributed.html

## License

MIT

---

**Last Updated**: 2025-11-09
**Dagu Version**: 1.24.0+
