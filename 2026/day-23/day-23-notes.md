# Day 23 – Git Branching & Working with GitHub
1. What is a branch in Git?
# Answer:  A branch in Git is a separate line of development. It allows developers to work on new features, bug fixes, or experiments without affecting the main project code.
# Each branch has its own commits and changes. By default, Git creates a branch called main (or sometimes master).
# Example:git branch feature-login
# This command creates a new branch called feature-login.

# 2. Why do we use branches instead of committing everything to main?
We use branches to keep the main branch stable and clean.
# Reasons:
Developers can work on new features safely
Prevents breaking the main production code
Allows multiple developers to work simultaneously
Makes testing and debugging easier
# Example workflow: main branch → stable production codefeature branch → new feature development
After testing, the feature branch is merged back into the main branch.

# 3. What is HEAD in Git?
#  HEAD is a pointer that refers to the current branch or the latest commit you are working on.
# In simple terms:
HEAD tells Git where you currently are in the repository.
# Example:
HEAD → main → latest commit
If you switch to another branch:
HEAD → feature-login → latest commit

# 4. What happens to your files when you switch branches?
When you switch branches, Git changes the files in your working directory to match the state of that branch.
This means:
Files may change
Some files may appear
Some files may disappear
# Example:
git checkout feature-login
Git updates the project files to match the feature-login branch.

# Task 2: Branching Commands — Hands-On List all branches
git branch
 Create new branch feature-1
git branch feature-1
 Switch to feature-1
git checkout feature-1
Ya modern command:
git switch feature-1
 Create new branch and switch in one command
git checkout -b feature-2
ya modern way:
git switch -c feature-2
 Move between branches
git switch feature-1
git switch main
📌 Difference:
Command	Purpose
git checkout	Old command – branch switch + file restore
git switch	New command – only branch switching
 Make commit on feature-1
git switch feature-1
echo "Feature 1 work" >> feature1.txt
git add feature1.txt
git commit -m "Added feature1 work"

Task 3: Push to GitHub
 Create repo on GitHub
GitHub → New repository
Name: devops-git-practice
README mat select karo
Connect local repo to GitHub
git remote add origin https://github.com/Devel955/devops-git-practice.git
Check:
git remote -v
 Push main branch
git push -u origin main
 Push feature-1 branch
git push -u origin feature-1
 Verify on GitHub
GitHub repo → Branches section
Tumhe dikhna chahiye:
main
feature-1
Answer: Origin vs Upstream
origin  → your forked repository (your GitHub repo)
upstream → original repository from which you forked
Example:
origin → github.com/Devel955/devops-git-practice
upstream → github.com/original-owner/devops-git-practice
 Switch back to main
git switch main
Check file:ls 
feature1.txt main branch me nahi hoga.
 Delete unused branch
git branch -d feature-2

Task 4: Pull from GitHub
GitHub par change karo
GitHub → edit file → commit
Local me update lo
git pull origin main
Difference: git fetch vs git pull
Command	Meaning
git fetch	Sirf changes download karta hai
git pull	Download + merge karta hai

Task 5: Clone vs Fork
Clone repo
git clone https://github.com/user/repo.git
Fork repo
GitHub → Fork button
Then clone fork:
git clone https://github.com/yourusername/repo.git
Answers (Notes me likhna)
Difference between Clone and Fork
Clone	Fork
Git command	GitHub feature
Local copy banata hai	GitHub account me copy banata hai
When to use
Clone → jab sirf project download karna ho
Fork → jab kisi open source project me contribute karna ho
Sync fork with original repo
git remote add upstream https://github.com/original-owner/repo.git
git fetch upstream
git merge upstream/main

