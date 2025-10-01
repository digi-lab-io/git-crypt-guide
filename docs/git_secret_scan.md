# digi-lab.io | Git Secret Scanning with Trivy v1.0.2

Before implementing git-crypt, it's essential to scan your repository for existing secrets that may have been accidentally committed. Trivy is an excellent tool for this purpose.

## 🚀 Quick Start with Docker

**Don't want to install anything? Start scanning immediately with Docker:**

```bash
# Scan current directory for secrets (easiest way to get started)
docker run --rm -v "$(pwd):/workspace" aquasec/trivy:latest fs --scanners secret .

# Windows PowerShell version
docker run --rm -v "${PWD}:/workspace" aquasec/trivy:latest fs --scanners secret .

# Scan with formatted output
docker run --rm -v "$(pwd):/workspace" aquasec/trivy:latest fs --scanners secret --format table .

# Save results to file
docker run --rm -v "$(pwd):/workspace" aquasec/trivy:latest fs --scanners secret --format json --output secrets-report.json .
```

> **💡 Pro Tip**: This Docker approach requires no installation and works immediately on any system with Docker!

## Table of Contents

- [Installing Trivy](#installing-trivy)
- [Scanning for Secrets](#scanning-for-secrets)
- [Custom Configuration](#custom-configuration)
- [Pre-commit Integration](#pre-commit-integration)
- [Remediation Workflow](#remediation-workflow)
- [Reporting and Monitoring](#reporting-and-monitoring)
- [Best Practices](#best-practices)

## Installing Trivy

### Linux/macOS

```bash
# Using curl
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Using Homebrew (macOS)
brew install trivy

# Using apt (Debian/Ubuntu)
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy
```

### Windows

```bash
# Using Chocolatey
choco install trivy

# Using Scoop
scoop install trivy
```

## Scanning for Secrets

### Basic Secret Scan

```bash
# Scan current repository for secrets
trivy fs --scanners secret .

# Scan specific directory
trivy fs --scanners secret /path/to/your/repo

# Output to file
trivy fs --scanners secret --format json --output secrets-report.json .
```

### Advanced Scanning Options

```bash
# Scan with severity filtering
trivy fs --scanners secret --severity HIGH,CRITICAL .

# Scan and ignore specific files/directories
trivy fs --scanners secret --skip-dirs node_modules,vendor .

# Scan with custom secret patterns
trivy fs --scanners secret --secret-config .trivy-secret.yaml .
```

## Custom Configuration

Create a `.trivy-secret.yaml` file for custom secret detection:

```yaml
# .trivy-secret.yaml
rules:
  - id: custom-api-key
    pattern: 'api[_-]?key\s*[:=]\s*["\']?([a-zA-Z0-9]{32,})["\']?'
    keywords:
      - api_key
      - apikey
    entropy: 3.5
    allow-list:
      - 'test_api_key_12345'
      - 'dummy_key_for_testing'
  
  - id: database-url
    pattern: 'DATABASE_URL\s*[:=]\s*["\']?(postgresql|mysql|mongodb)://[^\s"\']+["\']?'
    keywords:
      - DATABASE_URL
      - DB_URL
    entropy: 4.0
    
  - id: jwt-secret
    pattern: 'JWT_SECRET\s*[:=]\s*["\']?([a-zA-Z0-9+/]{32,}={0,2})["\']?'
    keywords:
      - JWT_SECRET
      - JWT_KEY
    entropy: 4.5
```

## Pre-commit Integration

Integrate Trivy into your development workflow to prevent secrets from being committed.

### Git Pre-commit Hook

```bash
#!/bin/bash
# .git/hooks/pre-commit

echo "🔍 Scanning for secrets before commit..."

# Run Trivy secret scan
if trivy fs --scanners secret --exit-code 1 --quiet .; then
    echo "✅ No secrets detected"
else
    echo "❌ Secrets detected! Commit blocked."
    echo "Please review and remove secrets, then encrypt with git-crypt if needed."
    exit 1
fi
```

Make the hook executable:

```bash
chmod +x .git/hooks/pre-commit
```

### GitHub Actions Integration

```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on: [push, pull_request]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Trivy secret scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scanners: 'secret'
          format: 'sarif'
          output: 'trivy-results.sarif'
          
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
          
      - name: Comment PR with results
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v6
        with:
          script: |
            const fs = require('fs');
            const results = JSON.parse(fs.readFileSync('trivy-results.sarif', 'utf8'));
            
            if (results.runs[0].results.length > 0) {
              github.rest.issues.createComment({
                issue_number: context.issue.number,
                owner: context.repo.owner,
                repo: context.repo.repo,
                body: '🚨 **Secrets detected in this PR!** Please review and remove before merging.'
              });
            }
```

### GitLab CI Integration

```yaml
# .gitlab-ci.yml
stages:
  - security

secret-scan:
  stage: security
  image: aquasec/trivy:latest
  script:
    - trivy fs --scanners secret --format template --template '@contrib/gitlab.tpl' --output gl-secret-scanning-report.json .
  artifacts:
    reports:
      secret_detection: gl-secret-scanning-report.json
  only:
    - merge_requests
    - main
```

## Remediation Workflow

When secrets are found, follow this systematic approach:

### 1. Immediate Actions

```bash
# View detailed scan results
trivy fs --scanners secret --format table .

# Get specific file locations
trivy fs --scanners secret --format json . | jq '.Results[].Secrets[].RuleID, .Results[].Target'

# Export findings for review
trivy fs --scanners secret --format json . > secret-findings.json
```

### 2. Assess Impact

```bash
# Check if secrets are in git history
git log --all --full-history -- path/to/secret/file

# Search for secret patterns in history
git log --all -S "your-secret-pattern" --source --all

# Check remote repositories
git remote -v
```

### 3. Clean Repository History

If secrets were committed to git history:

```bash
# Using git-filter-repo (recommended)
# Install: pip install git-filter-repo

# Remove specific files
git filter-repo --path-based-filter 'return b"secret-file.txt" not in path'

# Remove specific patterns
echo "password123" > expressions.txt
echo "api_key_secret" >> expressions.txt
git filter-repo --replace-text expressions.txt

# Remove by content pattern
git filter-repo --message-callback 'return message.replace(b"SECRET_KEY=", b"SECRET_KEY=[REDACTED]")'
```

Alternative using BFG Repo-Cleaner:

```bash
# Install BFG
# Download from: https://rtyley.github.io/bfg-repo-cleaner/

# Remove files
java -jar bfg.jar --delete-files secret-file.txt

# Remove patterns
echo "password123" > passwords.txt
java -jar bfg.jar --replace-text passwords.txt
```

### 4. Implement git-crypt

```bash
# Initialize git-crypt for future secrets
git-crypt init
git-crypt add-gpg-user your.email@domain.com

# Add patterns to .gitattributes
echo "*.secret filter=git-crypt diff=git-crypt" >> .gitattributes
echo "config/credentials.yml filter=git-crypt diff=git-crypt" >> .gitattributes
echo ".env.production filter=git-crypt diff=git-crypt" >> .gitattributes

# Commit the configuration
git add .gitattributes
git commit -m "Configure git-crypt encryption patterns"
```

### 5. Rotate Compromised Secrets

```bash
# Create a checklist script
#!/bin/bash
# rotate-secrets.sh

echo "🔄 Secret Rotation Checklist:"
echo "[ ] Database passwords"
echo "[ ] API keys"
echo "[ ] JWT secrets"
echo "[ ] SSL certificates"
echo "[ ] Cloud service credentials"
echo "[ ] Third-party service tokens"
echo "[ ] Webhook secrets"

# Add your rotation commands here
# aws secretsmanager update-secret --secret-id prod/db/password --secret-string "new-password"
# kubectl create secret generic app-secrets --from-literal=api-key="new-key" --dry-run=client -o yaml | kubectl apply -f -
```

## Reporting and Monitoring

### Generate Comprehensive Reports

```bash
# Generate HTML report
trivy fs --scanners secret --format template --template '@contrib/html.tpl' --output secret-report.html .

# Generate summary report
trivy fs --scanners secret --format table --output secret-summary.txt .

# Generate CSV for spreadsheet analysis
trivy fs --scanners secret --format json . | jq -r '.Results[]?.Secrets[]? | [.RuleID, .Category, .Severity, .Title, .StartLine, .EndLine] | @csv' > secrets.csv
```

### Continuous Monitoring Script

```bash
#!/bin/bash
# monitor-secrets.sh

# Configuration
REPO_PATH=${1:-.}
REPORTS_DIR="security-reports"
DATE=$(date +%Y%m%d_%H%M%S)
REPORT_FILE="${REPORTS_DIR}/secrets_${DATE}.json"

# Create reports directory
mkdir -p "${REPORTS_DIR}"

echo "🔍 Starting secret scan for: ${REPO_PATH}"

# Run Trivy scan
trivy fs --scanners secret --format json --output "${REPORT_FILE}" "${REPO_PATH}"
SCAN_EXIT_CODE=$?

# Generate summary
SECRETS_COUNT=$(jq '[.Results[]?.Secrets[]?] | length' "${REPORT_FILE}" 2>/dev/null || echo "0")

if [ "${SCAN_EXIT_CODE}" -eq 0 ] && [ "${SECRETS_COUNT}" -eq 0 ]; then
    echo "✅ Secret scan completed: No secrets detected"
    echo "📄 Report saved: ${REPORT_FILE}"
else
    echo "❌ Secrets detected! Count: ${SECRETS_COUNT}"
    echo "📄 Detailed report: ${REPORT_FILE}"
    
    # Generate alert summary
    jq -r '.Results[]?.Secrets[]? | "⚠️  \(.RuleID): \(.Title) in \(.Target):\(.StartLine)"' "${REPORT_FILE}" | head -10
    
    # Send notifications (customize as needed)
    if command -v mail >/dev/null 2>&1; then
        echo "Secrets detected in repository scan. See attached report." | mail -s "Security Alert: Secrets Detected" -a "${REPORT_FILE}" security@company.com
    fi
    
    # Slack notification (requires webhook URL)
    if [ -n "${SLACK_WEBHOOK_URL}" ]; then
        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"🚨 Secret scan alert: ${SECRETS_COUNT} secrets detected in ${REPO_PATH}\"}" \
            "${SLACK_WEBHOOK_URL}"
    fi
fi

# Cleanup old reports (keep last 30 days)
find "${REPORTS_DIR}" -name "secrets_*.json" -mtime +30 -delete

echo "🏁 Secret monitoring completed"
exit ${SCAN_EXIT_CODE}
```

Make it executable and run:

```bash
chmod +x monitor-secrets.sh
./monitor-secrets.sh /path/to/your/repo
```

### Scheduled Monitoring with Cron

```bash
# Add to crontab for daily scans
# crontab -e

# Run secret scan daily at 2 AM
0 2 * * * /path/to/monitor-secrets.sh /path/to/your/repo >> /var/log/secret-scan.log 2>&1

# Run secret scan on all repositories weekly
0 2 * * 0 find /path/to/repos -type d -name ".git" -exec dirname {} \; | xargs -I {} /path/to/monitor-secrets.sh {}
```

## Best Practices

### Development Workflow

1. **Pre-commit Scanning**: Always scan before committing

   ```bash
   # Add to your development workflow
   trivy fs --scanners secret --quiet . && git commit -m "Your commit message"
   ```

2. **Branch Protection**: Use GitHub/GitLab branch protection with required status checks

3. **Developer Education**: Train team members on secret detection

   ```bash
   # Create a quick check alias
   echo 'alias secret-check="trivy fs --scanners secret --quiet ."' >> ~/.bashrc
   ```

### CI/CD Integration

1. **Fail Fast**: Configure pipelines to fail on secret detection
2. **Multiple Scanners**: Use Trivy alongside other tools (gitleaks, detect-secrets)
3. **Staged Scanning**: Scan at multiple pipeline stages

### Incident Response

1. **Response Plan**: Have a documented process for secret discoveries
2. **Communication**: Establish notification channels
3. **Rotation Procedures**: Automate secret rotation where possible
4. **Post-Incident**: Document lessons learned and improve processes

### False Positive Management

1. **Allow-lists**: Maintain lists of known test data and false positives
2. **Custom Rules**: Create specific patterns for your codebase
3. **Regular Review**: Periodically review and update configurations

### Team Practices

1. **Code Reviews**: Include secret scanning in review processes
2. **Documentation**: Maintain secret handling guidelines
3. **Training**: Regular security awareness sessions
4. **Tools**: Provide developers with easy-to-use scanning tools

## Integration with Git-Crypt

This secret scanning approach works perfectly with git-crypt:

1. **Pre-implementation**: Scan and clean your repository before setting up git-crypt
2. **Ongoing Protection**: Continue scanning to catch new secrets before encryption
3. **Migration Path**: Use findings to determine what should be encrypted with git-crypt
4. **Compliance**: Maintain audit trails of secret handling

Remember: **Prevention is better than detection**. While secret scanning is crucial, the best approach is to never commit secrets in the first place by using proper environment variable management and git-crypt encryption.
