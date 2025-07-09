# Useful Git Commands

## Basic Commands

### Initialization and Setup
```bash
# Initialize a new git repository
git init

# Clone a repository
git clone <repository-url>

# Configure user information
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

# View current configuration
git config --list
```

### Basic Workflow
```bash
# Check repository status
git status

# Add files to staging area
git add <filename>          # Add specific file
git add .                   # Add all files
git add *.js               # Add all JavaScript files

# Commit changes
git commit -m "Your commit message"
git commit -am "Message"    # Add and commit tracked files

# View commit history
git log
git log --oneline          # Compact view
git log --graph --oneline  # Graph view
```

## Branching and Merging

### Branch Management
```bash
# List all branches
git branch

# Create and switch to new branch
git checkout -b <branch-name>
git switch -c <branch-name>  # Modern syntax

# Switch between branches
git checkout <branch-name>
git switch <branch-name>      # Modern syntax

# Delete branch
git branch -d <branch-name>  # Safe delete
git branch -D <branch-name>  # Force delete

# Rename current branch
git branch -m <new-name>
```

### Merging
```bash
# Merge branch into current branch
git merge <branch-name>

# Merge with no fast-forward (always create merge commit)
git merge --no-ff <branch-name>

# Abort merge if conflicts occur
git merge --abort
```

## Remote Repository

### Remote Operations
```bash
# Add remote repository
git remote add origin <repository-url>

# List remote repositories
git remote -v

# Fetch changes from remote
git fetch origin

# Pull changes from remote
git pull origin <branch-name>
git pull                    # Pull from current branch

# Push changes to remote
git push origin <branch-name>
git push                    # Push to current branch

# Push new branch to remote
git push -u origin <branch-name>
```

## Advanced Commands

### Stashing
```bash
# Save changes temporarily
git stash
git stash save "Message"

# List stashes
git stash list

# Apply most recent stash
git stash pop

# Apply specific stash
git stash apply stash@{n}

# Drop stash
git stash drop stash@{n}

# Clear all stashes
git stash clear
```

### Reset and Revert
```bash
# Soft reset (keep changes in staging)
git reset --soft HEAD~1

# Mixed reset (keep changes unstaged)
git reset HEAD~1

# Hard reset (discard all changes)
git reset --hard HEAD~1

# Reset to specific commit
git reset --hard <commit-hash>

# Revert last commit (creates new commit)
git revert HEAD
```

### Cherry-pick and Rebase
```bash
# Apply specific commit to current branch
git cherry-pick <commit-hash>

# Interactive rebase
git rebase -i HEAD~n

# Rebase current branch onto another
git rebase <base-branch>
```

## Information and Debugging

### Viewing Information
```bash
# Show differences
git diff                    # Working directory vs staging
git diff --staged          # Staging vs last commit
git diff HEAD~1            # Current vs previous commit

# Show file history
git log --follow <filename>

# Show who changed what
git blame <filename>

# Show commit details
git show <commit-hash>
```

### Cleanup and Maintenance
```bash
# Remove untracked files
git clean -n               # Dry run
git clean -f               # Force delete

# Prune remote references
git remote prune origin

# Garbage collection
git gc

# Check repository health
git fsck
```

## Aliases and Shortcuts

### Useful Aliases
```bash
# Add these to your ~/.gitconfig

[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    unstage = reset HEAD --
    last = log -1 HEAD
    visual = !gitk
    lg = log --oneline --graph --decorate
    ll = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
```

## Tips and Best Practices

### Commit Messages
- Use present tense ("Add feature" not "Added feature")
- Use imperative mood ("Move cursor" not "Moves cursor")
- Keep first line under 50 characters
- Separate subject from body with blank line
- Use body to explain what and why, not how

### Branch Naming
- `feature/feature-name` - New features
- `bugfix/bug-description` - Bug fixes
- `hotfix/urgent-fix` - Critical fixes
- `release/version` - Release preparation

### Workflow Tips
```bash
# Before starting work
git pull origin main

# Check what you're about to commit
git diff --staged

# Amend last commit (if no push yet)
git commit --amend

# Create backup branch before major changes
git checkout -b backup/feature-name
```

## Troubleshooting

### Common Issues
```bash
# Fix detached HEAD
git checkout main

# Recover deleted branch
git reflog
git checkout -b <branch-name> <commit-hash>

# Undo last commit (keep changes)
git reset --soft HEAD~1

# Find lost commits
git reflog

# Clean up local branches that no longer exist on remote
git remote prune origin
git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -D
```

## Advanced Git Features

### Submodules
```bash
# Add submodule
git submodule add <repository-url> <path>

# Initialize submodules
git submodule init
git submodule update

# Update all submodules
git submodule update --remote
```

### Git Hooks
```bash
# Pre-commit hook example (save as .git/hooks/pre-commit)
#!/bin/sh
# Run tests before commit
npm test
```

### Git Worktree
```bash
# Create additional working tree
git worktree add ../path-to-worktree branch-name

# List worktrees
git worktree list

# Remove worktree
git worktree remove path-to-worktree
```

## Git Ignore Patterns

### Common .gitignore entries
```
# Dependencies
node_modules/
vendor/

# Build outputs
dist/
build/
*.o
*.so

# IDE files
.vscode/
.idea/
*.swp
*.swo

# OS files
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Environment variables
.env
.env.local
```

## Git Configuration

### Global Configuration
```bash
# Set default editor
git config --global core.editor "code --wait"

# Set default branch name
git config --global init.defaultBranch main

# Set credential helper
git config --global credential.helper store

# Set merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

This comprehensive list covers most common git operations and should serve as a useful reference for your development workflow.
