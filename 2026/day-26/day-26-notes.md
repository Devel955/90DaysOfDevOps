# Go to your DevOps repo
Go to the repo where you keep your 90DaysOfDevOps / devops-git-practice work.
# Create the Day-26 folder
mkdir -p 2026/day-26
# Create the notes file
nano 2026/day-26/day-26-notes.md
Save:
CTRL + X
Y
Enter

# Update git-commands.md
Add these GitHub CLI commands to your file:
# GitHub CLI Commands

gh auth login
gh auth status

gh repo create
gh repo list
gh repo view
gh repo clone

gh issue create
gh issue list
gh issue view
gh issue close

gh pr create
gh pr list
gh pr view
gh pr merge

gh run list
gh run view

gh api
gh gist create

Add files to git
git add .
# Commit
git commit -m "Day 26 - GitHub CLI"
#  Push to GitHub
git push
# Final GitHub structure
Your repo should look like this:

devops-git-practice
│
├── git-commands.md
│
└── 2026
     │
     └── day-26
          │
          └── day-26-notes.md
gh release create
gh alias set
gh search repos
