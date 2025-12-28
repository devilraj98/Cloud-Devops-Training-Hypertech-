# Day 22: Git & GitHub - Advanced Features and Workflows

## 📚 Learning Objectives
- Master advanced Git commands and workflows
- Implement branching strategies (GitFlow, GitHub Flow)
- Practice code review processes and pull requests
- Understand GitHub Actions basics
- Learn repository management and collaboration

## 🌳 Git Fundamentals Review

### Git Architecture
```bash
Working Directory → Staging Area → Local Repository → Remote Repository
     (git add)        (git commit)      (git push)
```

### Essential Git Commands
```bash
# Repository initialization
git init
git clone <repository-url>

# Basic workflow
git add <file>
git commit -m "message"
git push origin <branch>
git pull origin <branch>

# Status and history
git status
git log --oneline
git diff
```

## 🔄 Advanced Git Workflows

### GitFlow Workflow

#### 1. Initialize GitFlow
```bash
# Install git-flow (if not installed)
# Ubuntu/Debian: sudo apt install git-flow
# macOS: brew install git-flow-avh

# Initialize GitFlow in repository
git flow init

# Default branch names:
# - main/master: production branch
# - develop: integration branch
# - feature/: feature branches
# - release/: release branches
# - hotfix/: hotfix branches
```

#### 2. Feature Development
```bash
# Start new feature
git flow feature start user-authentication

# Work on feature
git add .
git commit -m "Add user login functionality"
git commit -m "Add password validation"

# Finish feature (merges to develop)
git flow feature finish user-authentication

# Publish feature for collaboration
git flow feature publish user-authentication
```

#### 3. Release Management
```bash
# Start release branch
git flow release start v1.2.0

# Make release preparations
git add .
git commit -m "Bump version to 1.2.0"
git commit -m "Update changelog"

# Finish release (merges to main and develop)
git flow release finish v1.2.0
```

#### 4. Hotfix Process
```bash
# Start hotfix from main
git flow hotfix start critical-security-fix

# Fix the issue
git add .
git commit -m "Fix security vulnerability in auth module"

# Finish hotfix (merges to main and develop)
git flow hotfix finish critical-security-fix
```

### GitHub Flow (Simplified)

#### 1. Feature Branch Workflow
```bash
# Create and switch to feature branch
git checkout -b feature/add-payment-gateway
git push -u origin feature/add-payment-gateway

# Make changes and commit
git add .
git commit -m "Add Stripe payment integration"
git push origin feature/add-payment-gateway

# Create pull request via GitHub UI
# After review and approval, merge to main
```

## 🛠️ Hands-on Implementation

### Step 1: Set Up Repository Structure

#### 1.1 Create Project Repository
```bash
# Create new repository
mkdir devops-project
cd devops-project
git init

# Create initial structure
mkdir -p src/{frontend,backend,infrastructure}
mkdir -p docs/{api,deployment}
mkdir -p tests/{unit,integration}
mkdir .github/workflows

# Create README
cat > README.md << 'EOF'
# DevOps Project

## Overview
This project demonstrates advanced Git workflows and CI/CD practices.

## Structure
- `src/frontend/` - Frontend application
- `src/backend/` - Backend API
- `src/infrastructure/` - Infrastructure as Code
- `docs/` - Documentation
- `tests/` - Test suites
- `.github/workflows/` - GitHub Actions workflows

## Getting Started
1. Clone the repository
2. Install dependencies
3. Run tests
4. Deploy to staging

## Contributing
Please read CONTRIBUTING.md for details on our code of conduct and the process for submitting pull requests.
EOF

# Create CONTRIBUTING.md
cat > CONTRIBUTING.md << 'EOF'
# Contributing Guidelines

## Branching Strategy
We use GitFlow for our branching strategy:
- `main` - Production-ready code
- `develop` - Integration branch
- `feature/*` - Feature development
- `release/*` - Release preparation
- `hotfix/*` - Critical fixes

## Pull Request Process
1. Create feature branch from develop
2. Make your changes
3. Write/update tests
4. Update documentation
5. Create pull request
6. Address review feedback
7. Merge after approval

## Code Standards
- Follow language-specific style guides
- Write meaningful commit messages
- Include tests for new features
- Update documentation
EOF

# Initial commit
git add .
git commit -m "Initial project structure"
```

#### 1.2 Create Sample Application
```bash
# Frontend application
cat > src/frontend/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>DevOps Demo App</title>
    <style>
        body { font-family: Arial; margin: 40px; background: #f0f0f0; }
        .container { background: white; padding: 30px; border-radius: 10px; }
        .version { color: #666; font-size: 12px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 DevOps Demo Application</h1>
        <p>This application demonstrates Git workflows and CI/CD practices.</p>
        <div class="version">Version: 1.0.0</div>
        <div id="api-status">Loading API status...</div>
    </div>
    
    <script>
        // Fetch API status
        fetch('/api/status')
            .then(response => response.json())
            .then(data => {
                document.getElementById('api-status').innerHTML = 
                    `<p>API Status: <strong>${data.status}</strong></p>`;
            })
            .catch(error => {
                document.getElementById('api-status').innerHTML = 
                    '<p>API Status: <strong>Offline</strong></p>';
            });
    </script>
</body>
</html>
EOF

# Backend API
cat > src/backend/app.py << 'EOF'
from flask import Flask, jsonify
from datetime import datetime
import os

app = Flask(__name__)

@app.route('/api/status')
def status():
    """API health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'version': '1.0.0',
        'environment': os.environ.get('ENVIRONMENT', 'development')
    })

@app.route('/api/users')
def users():
    """Sample users endpoint"""
    return jsonify({
        'users': [
            {'id': 1, 'name': 'Alice', 'role': 'admin'},
            {'id': 2, 'name': 'Bob', 'role': 'user'},
            {'id': 3, 'name': 'Charlie', 'role': 'user'}
        ]
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF

# Requirements file
cat > src/backend/requirements.txt << 'EOF'
Flask==2.3.3
gunicorn==21.2.0
pytest==7.4.2
EOF

# Dockerfile
cat > src/backend/Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
EOF

git add .
git commit -m "Add sample frontend and backend applications"
```

### Step 2: Implement Advanced Git Features

#### 2.1 Interactive Rebase
```bash
# Create multiple commits for demonstration
echo "# API Documentation" > docs/api/README.md
git add docs/api/README.md
git commit -m "Add API documentation"

echo "## Endpoints" >> docs/api/README.md
git add docs/api/README.md
git commit -m "Document API endpoints"

echo "## Authentication" >> docs/api/README.md
git add docs/api/README.md
git commit -m "Add authentication docs"

# Interactive rebase to squash commits
git rebase -i HEAD~3

# In the editor, change 'pick' to 'squash' for commits 2 and 3:
# pick abc1234 Add API documentation
# squash def5678 Document API endpoints
# squash ghi9012 Add authentication docs
```

#### 2.2 Git Hooks
```bash
# Create pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash

echo "Running pre-commit checks..."

# Check for Python syntax errors
if find . -name "*.py" -exec python -m py_compile {} \; 2>/dev/null; then
    echo "✅ Python syntax check passed"
else
    echo "❌ Python syntax errors found"
    exit 1
fi

# Check for TODO comments in staged files
if git diff --cached --name-only | xargs grep -l "TODO" 2>/dev/null; then
    echo "⚠️  Warning: TODO comments found in staged files"
    echo "Consider addressing them before committing"
fi

# Check commit message format (if available)
if [ -f .gitmessage ]; then
    echo "✅ Using commit message template"
fi

echo "Pre-commit checks completed"
EOF

chmod +x .git/hooks/pre-commit

# Create commit message template
cat > .gitmessage << 'EOF'
# <type>: <subject>
#
# <body>
#
# <footer>

# Type should be one of the following:
# * feat (new feature)
# * fix (bug fix)
# * docs (documentation)
# * style (formatting, missing semi colons, etc)
# * refactor (refactoring production code)
# * test (adding tests, refactoring test)
# * chore (updating build tasks, package manager configs, etc)
EOF

git config commit.template .gitmessage
```

#### 2.3 Git Aliases
```bash
# Set up useful Git aliases
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual '!gitk'
git config --global alias.graph 'log --oneline --graph --decorate --all'
git config --global alias.amend 'commit --amend --no-edit'

# Advanced aliases
git config --global alias.find-merge 'log --merges --oneline'
git config --global alias.contributors 'shortlog -sn'
git config --global alias.cleanup 'branch --merged | grep -v "\*\|main\|develop" | xargs -n 1 git branch -d'
```

### Step 3: GitHub Integration

#### 3.1 Create GitHub Repository
```bash
# Create repository on GitHub (via web interface or CLI)
gh repo create devops-project --public --description "DevOps project demonstrating Git workflows"

# Add remote and push
git remote add origin https://github.com/yourusername/devops-project.git
git branch -M main
git push -u origin main

# Create develop branch
git checkout -b develop
git push -u origin develop
```

#### 3.2 Branch Protection Rules
```bash
# Configure branch protection via GitHub web interface:
# Settings → Branches → Add rule

# Protection settings for main branch:
# - Require pull request reviews before merging
# - Require status checks to pass before merging
# - Require branches to be up to date before merging
# - Include administrators
# - Restrict pushes that create files larger than 100MB
```

#### 3.3 Issue and Pull Request Templates
```bash
# Create issue template
mkdir -p .github/ISSUE_TEMPLATE

cat > .github/ISSUE_TEMPLATE/bug_report.md << 'EOF'
---
name: Bug report
about: Create a report to help us improve
title: '[BUG] '
labels: bug
assignees: ''
---

**Describe the bug**
A clear and concise description of what the bug is.

**To Reproduce**
Steps to reproduce the behavior:
1. Go to '...'
2. Click on '....'
3. Scroll down to '....'
4. See error

**Expected behavior**
A clear and concise description of what you expected to happen.

**Screenshots**
If applicable, add screenshots to help explain your problem.

**Environment:**
 - OS: [e.g. iOS]
 - Browser [e.g. chrome, safari]
 - Version [e.g. 22]

**Additional context**
Add any other context about the problem here.
EOF

cat > .github/ISSUE_TEMPLATE/feature_request.md << 'EOF'
---
name: Feature request
about: Suggest an idea for this project
title: '[FEATURE] '
labels: enhancement
assignees: ''
---

**Is your feature request related to a problem? Please describe.**
A clear and concise description of what the problem is.

**Describe the solution you'd like**
A clear and concise description of what you want to happen.

**Describe alternatives you've considered**
A clear and concise description of any alternative solutions or features you've considered.

**Additional context**
Add any other context or screenshots about the feature request here.
EOF

# Create pull request template
cat > .github/pull_request_template.md << 'EOF'
## Description
Brief description of changes made.

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Code is commented where necessary
- [ ] Documentation updated
- [ ] No new warnings introduced

## Screenshots (if applicable)
Add screenshots to help explain your changes.

## Related Issues
Closes #(issue number)
EOF

git add .github/
git commit -m "Add GitHub templates for issues and pull requests"
git push origin develop
```

## 🔄 Advanced Git Operations

### Stashing and Cherry-picking
```bash
# Stash changes
git stash push -m "Work in progress on user authentication"
git stash list
git stash apply stash@{0}
git stash drop stash@{0}

# Cherry-pick specific commits
git cherry-pick <commit-hash>
git cherry-pick --no-commit <commit-hash>  # Don't auto-commit
```

### Submodules
```bash
# Add submodule
git submodule add https://github.com/user/shared-library.git lib/shared

# Initialize submodules after cloning
git submodule init
git submodule update

# Update submodules
git submodule update --remote
```

### Advanced Merging
```bash
# Merge with custom strategy
git merge --strategy=ours feature-branch
git merge --strategy-option=theirs feature-branch

# Merge without fast-forward
git merge --no-ff feature-branch

# Squash merge
git merge --squash feature-branch
```

## 🧪 Testing Git Workflows

### Test 1: Feature Development Workflow
```bash
# Start feature branch
git checkout develop
git checkout -b feature/user-profile

# Make changes
echo "User profile functionality" > src/backend/profile.py
git add src/backend/profile.py
git commit -m "feat: add user profile management"

# Push and create PR
git push -u origin feature/user-profile
# Create pull request via GitHub UI
```

### Test 2: Code Review Process
```bash
# Reviewer checks out PR branch
git fetch origin
git checkout feature/user-profile

# Review changes
git diff develop..feature/user-profile
git log develop..feature/user-profile --oneline

# Test locally
cd src/backend
python -m pytest tests/

# Provide feedback via GitHub PR interface
```

### Test 3: Release Process
```bash
# Create release branch
git checkout develop
git checkout -b release/v1.1.0

# Update version numbers
sed -i 's/1.0.0/1.1.0/g' src/frontend/index.html
sed -i 's/1.0.0/1.1.0/g' src/backend/app.py

git add .
git commit -m "chore: bump version to 1.1.0"

# Merge to main
git checkout main
git merge --no-ff release/v1.1.0
git tag -a v1.1.0 -m "Release version 1.1.0"

# Merge back to develop
git checkout develop
git merge --no-ff release/v1.1.0

# Push everything
git push origin main develop --tags
```

## 📚 Interview Questions & Answers

### Fresher Level (1-10)

**Q1: What is the difference between git merge and git rebase?**
A: Git merge creates a merge commit combining two branches, preserving history. Git rebase replays commits from one branch onto another, creating a linear history without merge commits.

**Q2: What is GitFlow workflow?**
A: GitFlow is a branching model with main (production), develop (integration), feature/* (new features), release/* (release preparation), and hotfix/* (critical fixes) branches.

**Q3: How do you undo the last commit?**
A: Use `git reset --soft HEAD~1` to keep changes staged, `git reset --hard HEAD~1` to discard changes, or `git revert HEAD` to create a new commit that undoes the last one.

**Q4: What is a pull request?**
A: A pull request is a method of submitting contributions to a project. It allows developers to notify team members about changes, discuss modifications, and review code before merging.

**Q5: How do you resolve merge conflicts?**
A: Edit conflicted files to resolve conflicts, remove conflict markers (<<<<<<<, =======, >>>>>>>), stage resolved files with `git add`, then complete merge with `git commit`.

### Intermediate Level (6-15)

**Q6: What are Git hooks and how do you use them?**
A: Git hooks are scripts that run automatically at certain points in Git workflow. Examples: pre-commit (before commits), post-receive (after pushes). Used for code quality checks, automated testing, and deployment.

**Q7: How do you implement a code review process?**
A: Use feature branches, create pull requests, require approvals before merging, implement automated checks (CI/CD), use branch protection rules, and establish review guidelines and standards.

**Q8: What is the difference between GitHub Flow and GitFlow?**
A: GitHub Flow is simpler with main branch and feature branches only. GitFlow is more complex with multiple branch types (main, develop, feature, release, hotfix) suitable for scheduled releases.

**Q9: How do you handle large files in Git?**
A: Use Git LFS (Large File Storage) for files >100MB, add file patterns to .gitattributes, avoid committing large binaries, use external storage for assets, and implement proper .gitignore rules.

**Q10: What are Git submodules and when to use them?**
A: Git submodules allow including other Git repositories as subdirectories. Use for shared libraries, external dependencies, or when you need to include another project while maintaining separate version control.

## 🔑 Key Takeaways

- **Branching Strategy**: Choose appropriate workflow (GitFlow vs GitHub Flow)
- **Code Review**: Essential for code quality and knowledge sharing
- **Automation**: Use hooks and GitHub Actions for quality gates
- **Collaboration**: Proper templates and documentation improve teamwork
- **History Management**: Keep clean, meaningful commit history
- **Security**: Implement branch protection and access controls

## 🚀 Next Steps

- Day 23: Code Quality and Testing
- Day 24: Hands-on Lab - Git Workflow Implementation
- Implement advanced GitHub Actions workflows

---

**Hands-on Completed:** ✅ Advanced Git Workflows, GitHub Integration, Code Review Process  
**Duration:** 3-4 hours  
**Difficulty:** Intermediate to Advanced