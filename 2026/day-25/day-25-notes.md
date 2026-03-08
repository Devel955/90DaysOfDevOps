# Day 25 – Git Reset vs Revert & Branching Strategies

# Task 1 – Git Reset
Created three commits
Commit A
Commit B
Commit C
git reset --soft HEAD~1
Command
git reset --soft HEAD~1
Result:
The last commit is removed.
Changes remain in the staging area.
You can recommit the changes.
Example:
Commit A
Commit B
(staged changes from C)
git reset --mixed HEAD~1
Command
git reset --mixed HEAD~1
Result:
The last commit is removed.
Changes remain in the working directory.
Changes are removed from the staging area.
Example:
Commit A
Commit B
(changes from C are unstaged)
git reset --hard HEAD~1
Command
git reset --hard HEAD~1
Result:
The last commit is removed.
Changes are permanently deleted.
Working directory is reset.
Example:
Commit A
Commit B

# Difference between --soft, --mixed, and --hard
| Option  | Commit History | Staging Area   | Working Directory |
| ------- | -------------- | -------------- | ----------------- |
| --soft  | Reset          | Keeps changes  | Keeps changes     |
| --mixed | Reset          | Clears staging | Keeps changes     |
| --hard  | Reset          | Clears staging | Deletes changes   |

# Which one is destructive?
git reset --hard
Reason:
It removes commits and deletes changes permanently.
When to use each
--soft
Used when you want to undo the last commit but keep changes staged.
--mixed
Used when you want to undo the commit and keep changes in the working directory.
--hard
Used when you want to completely discard commits and changes.
Should you use git reset on pushed commits?
No.
Reason:
It rewrites commit history and may break other collaborators' repositories.

# Task 2 – Git Revert
Example commits
Commit X
Commit Y
Commit Z
Command
git revert <commit-id-of-Y>
Result
Git creates a new commit that undoes the changes introduced by commit Y.
Example history:
Commit X
Commit Y
Commit Z
Revert "Commit Y"
Is commit Y still in history?
Yes.
The original commit remains in the history.

# Difference between git revert and git reset.
| Feature                  | git reset | git revert |
| ------------------------ | --------- | ---------- |
| Changes history          | Yes       | No         |
| Creates new commit       | No        | Yes        |
| Safe for shared branches | No        | Yes        |
# Why is revert safer?
Because it does not modify existing history and is safe for collaborative environments.
When to use revert vs reset
Use reset for local commits that have not been pushed.
Use revert for changes in shared repositories.

# Task 3 – Reset vs Revert Summary:
| Feature                     | git reset                         | git revert                           |
| --------------------------- | --------------------------------- | ------------------------------------ |
| What it does                | Moves the branch pointer backward | Creates a new commit to undo changes |
| Removes commit from history | Yes                               | No                                   |
| Safe for shared branches    | No                                | Yes                                  |
| When to use                 | Local mistakes                    | Shared repository mistakes           |

# Task 4 – Branching Strategies
GitFlow
How it works
main
develop
feature/*
release/*
hotfix/*
Flow

main
 |
develop
 |---- feature branches
 |
release
 |
hotfix
Used in
Large teams and enterprise projects.
Pros
Structured workflow
Stable releases
Cons
Complex workflow
Slower development
GitHub Flow
How it works
main
 |
feature branch
 |
pull request
 |
merge to main
Used in
Startups and web applications.
Pros
Simple workflow
Fast deployment
Cons
Limited control for complex release cycles
Trunk-Based Development
How it works
main (trunk)
 |
short-lived branches
 |
merge frequently
Used in
Companies like Google and Facebook.
Pros
Fast integration
Continuous delivery
Cons
Requires strong automated testing
Strategy Answers
Which strategy would you use for a startup shipping fast?
GitHub Flow
Reason: Simple workflow and fast releases.
Which strategy would you use for a large team with scheduled releases?
GitFlow
Reason: Structured branching and controlled release cycles.
Example open-source project
Example: React
React follows a workflow similar to GitHub Flow using pull requests and the main branch.
# Task 5 – Git Commands Reference (Days 22–25)
Setup & Config
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
Basic Workflow
git init
git status
git add .
git commit -m "message"
git log
git diff
Branching
git branch
git checkout branch-name
git switch branch-name
Remote
git clone
git push
git pull
git fetch
Merge & Rebase
git merge branch-name
git rebase main
Stash & Cherry Pick
git stash
git stash pop
git cherry-pick <commit-id>
Reset & Revert
git reset --soft HEAD~1
git reset --mixed HEAD~1
git reset --hard HEAD~1
git revert <commit-id>
Safety Command
git reflog
Shows the complete history of Git actions and helps recover commits even after a hard reset.


