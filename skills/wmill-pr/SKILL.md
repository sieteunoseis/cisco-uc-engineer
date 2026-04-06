---
name: wmill-pr
description: Open a draft pull request on GitHub for Windmill repos. Use when creating or opening a PR.
license: MIT
metadata:
  author: windmill-labs
  version: "1.0.0"
---

# Pull Request Skill

Create a draft pull request with a clear title and explicit description of changes.

## Instructions

1. **Analyze branch changes**: Understand all commits since diverging from main
2. **Push to remote**: Ensure all commits are pushed
3. **Create draft PR**: Always open as draft for review before merging

## PR Title Format

Follow conventional commit format for the PR title:

```
<type>: <description>
```

### Types

- `feat`: New feature or capability
- `fix`: Bug fix
- `refactor`: Code restructuring
- `docs`: Documentation changes
- `chore`: Maintenance tasks
- `perf`: Performance improvements

### Title Rules

- Keep under 70 characters
- Use lowercase, imperative mood
- No period at the end
- If `*_ee.rs` files were modified, prefix with `[ee]`: `[ee] <type>: <description>`

## PR Body Format

```markdown
## Summary

<Clear description of what this PR does and why>

## Changes

- <Specific change 1>
- <Specific change 2>
- <Specific change 3>

## Test plan

- [ ] <How to verify change 1>
- [ ] <How to verify change 2>

---

Generated with [Claude Code](https://claude.com/claude-code)
```

## Execution Steps

1. Run `git status` to check for uncommitted changes
2. Run `git log main..HEAD --oneline` to see all commits in this branch
3. Run `git diff main...HEAD` to see the full diff against main
4. Check if remote branch exists and is up to date
5. Push to remote if needed: `git push -u origin HEAD`
6. Create draft PR using gh CLI
7. Return the PR URL to the user

## EE Companion PR (when `*_ee.rs` files were modified)

The `*_ee.rs` files in the windmill repo are **symlinks** to `windmill-ee-private`. Check the EE repo for uncommitted or unpushed changes and follow the EE PR workflow in `docs/enterprise.md`.
