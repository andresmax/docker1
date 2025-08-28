# Kamal Deployment to DigitalOcean - Hybrid Manual/Automatic Setup

## Overview
Rails-approved deployment that supports BOTH automatic deploys on push to main AND manual deploys via `kamal deploy`, with full team credential safety.

## Dual Deployment Strategy

### 1. Automatic Deployment (Push to Main)
GitHub Actions triggers on every push to main branch, using GitHub's built-in secrets.

### 2. Manual Deployment (kamal deploy)
Any authorized developer can deploy from their local machine using environment variables.

## Implementation Plan

### Step 1: GitHub Actions Setup
Create `.github/workflows/deploy.yml`:
```yaml
name: Deploy to Production
on:
  push:
    branches: [main]
  workflow_dispatch:  # Also allows manual trigger from GitHub UI

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: ruby/setup-ruby@v1
        with:
          ruby-version: .ruby-version
          
      - name: Install Kamal
        run: gem install kamal
        
      - name: Deploy to DigitalOcean
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.GITHUB_TOKEN }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
          DOCKER1_DATABASE_PASSWORD: ${{ secrets.DOCKER1_DATABASE_PASSWORD }}
        run: |
          kamal deploy
```

### Step 2: GitHub Repository Secrets
In GitHub repo settings → Secrets → Actions, add:
- `RAILS_MASTER_KEY` - Your production Rails master key
- `DOCKER1_DATABASE_PASSWORD` - Production database password
- Note: `GITHUB_TOKEN` is automatic, no need to add

### Step 3: Local Manual Deployment Setup
For developers who need to deploy manually:

Create `.env.production.local` (gitignored):
```bash
export KAMAL_REGISTRY_PASSWORD="github-personal-access-token"
export RAILS_MASTER_KEY="same-key-as-in-github-secrets"
export DOCKER1_DATABASE_PASSWORD="same-as-in-github-secrets"
```

Deploy manually:
```bash
source .env.production.local
kamal deploy
```

### Step 4: Update config/deploy.yml
```yaml
service: docker1
image: ghcr.io/your-org/docker1

servers:
  web:
    - your.droplet.ip

registry:
  server: ghcr.io
  username: your-github-username
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  secret:
    - RAILS_MASTER_KEY
    - DOCKER1_DATABASE_PASSWORD
  clear:
    SOLID_QUEUE_IN_PUMA: true

proxy:
  ssl: true
  host: yourdomain.com
```

## How It Works

### Automatic Deploy Flow:
1. Developer pushes to main
2. GitHub Actions runs automatically
3. Uses GitHub Secrets for credentials
4. Deploys to DigitalOcean
5. Team gets notified (optional: add Slack notification)

### Manual Deploy Flow:
1. Developer sources local env file: `source .env.production.local`
2. Runs: `kamal deploy`
3. Uses local ENV variables
4. Deploys to same DigitalOcean server

## Team Scenarios

### Scenario A: Junior Developer
- Can't deploy (no credentials)
- Pushes to feature branch
- Senior dev reviews and merges to main
- GitHub Actions auto-deploys

### Scenario B: Senior Developer
- Has credentials in `.env.production.local`
- Can merge to main (auto-deploy)
- OR can run `kamal deploy` for hotfixes
- OR can deploy specific branches manually

### Scenario C: Emergency Hotfix
- Senior dev creates hotfix branch
- Tests locally
- Deploys directly: `kamal deploy`
- Then merges to main for record

## Safety Features

1. **Credential Protection**
   - GitHub Secrets for CI/CD
   - Local .env files (gitignored) for manual
   - Never committed to repo

2. **Deployment Control**
   - Main branch protected (requires PR reviews)
   - Manual deploy requires credentials
   - All deploys use same configuration

3. **Rollback Capability**
   ```bash
   # Both methods can rollback
   kamal rollback  # Manual
   # OR revert commit and push to main (auto)
   ```

## File Organization
```
.github/workflows/deploy.yml     # Auto-deploy config (committed)
.env.production.local            # Manual deploy secrets (gitignored)
.kamal/secrets                   # ENV template (committed)
config/deploy.yml                # Kamal config (committed)
config/master.key                # Never committed (gitignored)
```

## Advantages of This Hybrid Approach

1. **Flexibility** - Deploy automatically OR manually as needed
2. **Security** - Credentials never in code
3. **Auditability** - GitHub Actions logs all auto-deploys
4. **Emergency Access** - Senior devs can deploy immediately
5. **Team Scaling** - Junior devs safe, senior devs empowered
6. **Single Source of Truth** - Same deploy.yml for both methods

## Setup Checklist

- [ ] Add GitHub Actions workflow file
- [ ] Set GitHub repository secrets
- [ ] Update config/deploy.yml with server details
- [ ] Create .env.production.local for manual deployers
- [ ] Test manual deploy
- [ ] Test auto-deploy by pushing to main
- [ ] Document credential sharing process for team

## Prerequisites

### DigitalOcean Droplet
- Create a $12-18/month droplet (2GB RAM minimum for Rails + PostgreSQL)
- Ubuntu 24.04 LTS
- Add your SSH key during creation

### GitHub Account
- Your existing GitHub account
- Generate a Personal Access Token with `write:packages` scope
- Settings → Developer settings → Personal access tokens → Tokens (classic)

### Domain Name
- Point your domain's A record to the droplet IP

## Database Strategy (Simplest)
- Use PostgreSQL on the same droplet initially
- Uncomment the `accessories/db` section in deploy.yml
- Migrate to DigitalOcean's managed PostgreSQL when scaling

## First Deployment Commands
```bash
# One-time setup
kamal setup

# Create and migrate database
kamal app exec 'bin/rails db:create'
kamal app exec 'bin/rails db:migrate'

# Subsequent deployments
kamal deploy
```

## Post-Deployment
- Monitor with: `kamal logs`
- Rails console: `kamal console`
- Check app health: `curl https://yourdomain.com`
- View container image at: `github.com/your-username/docker1/pkgs/container/docker1`

This gives you the best of both worlds: convenience of auto-deploy with the control of manual deployment when needed.