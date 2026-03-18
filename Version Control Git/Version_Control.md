# Version Control with Git & GitHub

---

## 1. The Pre-Version Control Problem

Before version control systems existed, developers managed file history manually — a messy, error-prone process.

### ▶ What developers used to do

- Manually save copies of files with names like `add_to_cart_final.apex`, `add_to_cart_after_uat.apex`, `add_to_cart_after_prod_20240616.apex`
- Each developer kept their own local copy of changed files
- No central system to track who changed what, or when

### ▶ Problems caused

- Changes made by one developer were invisible to others
- Overwrites and conflicts caused silent data loss
- No audit trail — impossible to trace bugs to their source
- Blame games started when bugs appeared — everyone thought their changes were fine

> 💡 **Key Pain Point:** If you had a bug and two developers each believed their code was correct, there was no reliable way to determine who introduced the issue.

---

## 2. What is Version Control?

Version Control is a system that tracks changes to files over time — who changed what, when, and why. It applies to code, documents, metadata, flows, and any files.

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Collaboration** | Multiple developers can work on the same file simultaneously without overwriting each other |
| **History Tracking** | Every change is logged with author, timestamp, and commit message |
| **Branching & Merging** | Each developer works in an isolated branch (playground) then merges into the main codebase |
| **Conflict Resolution** | When two developers edit the same lines, Git highlights the conflict and guides resolution |
| **Backup & Recovery** | Restore any file or project to any previous state — disaster recovery built in |
| **Blame / Audit Trail** | See exactly who changed which line and when — eliminates blame games |
| **CI/CD Integration** | Version control systems power continuous integration and deployment pipelines |
| **Security & Transparency** | All changes are logged, reviewable, and traceable |

---

## 3. Git vs GitHub — Key Difference

| Aspect | Details |
|--------|---------|
| **Git** | A distributed Version Control System (VCS) installed on your local machine. Tracks all changes, manages branches, handles merges. |
| **GitHub** | A cloud-based platform that stores your Git repositories online. Owned by Microsoft. Provides UI for history, PRs, code review, collaboration. |
| **Alternatives to GitHub** | GitLab, Bitbucket, Azure DevOps, Beanstalk — all work with Git using the same commands |
| **GitHub Actions** | GitHub's built-in CI/CD tool — separate from core GitHub. Used for automated deployment pipelines. |

> ⚠️ **Common misconception:** Git ≠ GitHub. Git is the engine; GitHub is just one of many garages where you park your code.

---

## 4. Setup — Step by Step

### Step 1: Create a GitHub Account

- Go to **github.com** → Sign Up
- Choose a memorable username (it appears in your commit history)
- Select the **Free plan** (sufficient for all personal and team work)

---

### Step 2: Generate a Personal Access Token

GitHub requires a token instead of a password for command-line access.

- Go to: **Profile → Settings → Developer Settings → Personal Access Tokens → Tokens (classic)**
- Or use the fine-grained tokens (beta) for more granular control
- Set expiry (recommended: 90 days or custom date)
- Select `repo` scope at minimum (gives full repository read/write access)
- Click **Generate Token** — COPY IT IMMEDIATELY (it is never shown again)

> 🔑 When Git asks for a password on the command line, enter your **Personal Access Token** — not your GitHub account password.

---

### Step 3: Install Git

- **Windows:** Download from [git-scm.com/downloads](https://git-scm.com/downloads)
- **macOS:** Use Homebrew — run: `brew install git`
- **Verify installation:** run `git --version` in terminal

```bash
git --version    # Should print: git version 2.x.x
```

---

### Step 4: Configure Git with Your Identity

This is mandatory. Your name and email are stamped on every commit.

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
```

> The email must match your GitHub account email so commits are linked to your GitHub profile.

---

### Step 5: Optional — Install GitHub Desktop

GitHub Desktop is a GUI alternative to the command line. Use it if you prefer a visual interface. It is not required if you use VS Code or the terminal.

---

## 5. Connecting VS Code to GitHub

### Case A: Clone an Existing Repository

Use this when joining an existing project that already lives on GitHub.

```bash
git clone <repository-url>
```

- Get the URL from GitHub: click **Code** button → **HTTPS** tab → copy the link
- Open Terminal inside your target folder, then run the clone command
- If prompted for credentials: username = GitHub username, password = your token
- After cloning, open the folder in VS Code: **File → Open Folder**

> 🔒 For private repositories, Git will prompt for credentials. Use your token as the password.

---

### Case B: Initialize a Brand-New Project

Use this when starting a fresh project locally and pushing it to GitHub for the first time.

- Open VS Code → click the **Source Control** icon (branch icon, left sidebar)
- Click **Initialize Repository** — this creates the hidden `.git` folder
- Stage all changes by clicking the `+` button next to `'Changes'`
- Write a commit message: `'Initial commit'`
- Click **Commit**, then **Publish Branch**
- Choose **Private** or **Public** repository → VS Code creates it on GitHub automatically

---

## 6. Branching — Your Personal Playground

**NEVER make changes directly on the `master` / `main` branch.** That branch represents production.

### Creating a Branch

- In VS Code: **Source Control → ··· (three dots) → Checkout To...**
- Type a new branch name following naming conventions
- Click **Create new Branch** or **Create new Branch from...** (choose source branch)

### ▶ Branch Naming Convention

| Type | Example Name |
|------|-------------|
| **New Feature** | `feature/add-to-cart` |
| **Bug Fix** | `bugfix/cart-total-error` |
| **Hotfix (production)** | `hotfix/payment-gateway` |
| **Release** | `release/v2.1.0` |

> 📌 Your new branch is **local-only** until you **Publish Branch**. No one else can see it until you push it.

---

## 7. Committing — Saving Your Work to Git

### The Commit Workflow

- Make changes to your files in VS Code
- Review changes: click the **Source Control** icon — you'll see all modified files
- Click the file to see a **diff** (left = old, right = new)
- Stage the files you want to commit: click the `+` icon next to each file
- Write a clear commit message describing what changed and why
- Click **Commit**

### ▶ Writing Good Commit Messages

> ✅ **Good:** `'Added add_to_cart and fetch_active_cart methods to CartService'`
>
> ❌ **Bad:** `'changes'`, `'fix'`, `'update'`

### Pushing to Remote

- After committing, VS Code shows `'Sync Changes'` or `'Publish Branch'` on the bottom bar
- **Sync Changes ↑** = push your local commits to GitHub
- The number next to the arrow = how many commits are waiting to be pushed

---

## 8. Pulling — Getting Others' Changes

Always pull before you start work if others share your branch.

```
Source Control → ··· (three dots) → Pull
```

- Fetches all commits from the remote branch into your local branch
- Prevents conflicts and keeps your code in sync with teammates
- **Best practice:** pull every morning before starting work

---

## 9. Pull Requests (PR) & Code Review

A **Pull Request** (also called Merge Request in GitLab) is a formal request to merge your feature branch into another branch (e.g., `master`). It enables code review before anything reaches production.

### Creating a Pull Request

- Go to your GitHub repository in the browser
- Click **Compare & Pull Request** (auto-suggested for your branch) OR click **New Pull Request**
- Set **Base branch** = the branch you want to merge INTO (e.g., `master`)
- Set **Compare branch** = YOUR branch (e.g., `feature/add-to-cart`)
- GitHub shows all commits and file changes for review
- Write a descriptive title and body — list what was changed and why
- Click **Create Pull Request**

### ▶ After Creating the PR

- Reviewers are assigned (team lead, architect, or designated reviewer)
- CI/CD pipeline may automatically validate the code against a sandbox org
- Reviewers can comment on specific lines, request changes, or approve
- Once approved: click **Merge Pull Request** (if you have permission)

> 🚦 As a developer, your job ends at **raising the PR**. Merging into `master` is the responsibility of the team lead / release manager.

---

## 10. Blame & History — Who Did What?

### Blame View *(on GitHub)*

- Open any file in GitHub → click the **Blame** tab
- Every line of code shows: who wrote it, when, and the commit message
- Click any commit hash to see the full set of changes in that commit

### Git Log *(in Terminal)*

```bash
git log    # Shows all commits with ID, author, date, and message
```

---

## 11. Reverting a Commit

If you need to undo a commit, use `git revert` (safe — creates a new undo commit rather than deleting history).

```bash
git log                      # Find the commit ID you want to undo
git revert <commit-id>       # Creates a new commit that reverses those changes
```

- After reverting, you must commit again to record the revert in history
- `git revert` is safer than `git reset` — it preserves full history

---

## 12. Forking — Contributing to Others' Repos

Forking creates a copy of someone else's repository under your own GitHub account. You can make changes freely, then raise a PR back to the original repo.

- Use forking to contribute to public or open-source projects
- After forking, clone **YOUR fork**, make changes, push to your fork
- Then raise a PR from your fork to the original repository
- Keep your fork in sync: use the **Sync Fork** button on GitHub

> 🔀 **Fork ≠ Clone.** A clone is a local copy of a repo. A fork is a remote copy of someone else's repo under your own GitHub account.

---

## 13. Recommended VS Code Extensions

| Extension | What It Does |
|-----------|-------------|
| **GitLens** | Shows inline blame info next to every line — author, date, and commit message. Extremely powerful for code review and audit. |
| **Git Graph** | Visual tree diagram of all commits and branches. Click any commit to see exact changes. Essential for understanding project history. |
| **Git History** | Browse file history, compare versions, and view previous content of any file. |
| **GitHub (built-in)** | Core VS Code GitHub integration — already installed by default. Provides the Source Control sidebar. |

---

## 14. Salesforce-Specific Notes

- Version control works for **ALL** Salesforce metadata: Apex classes, triggers, LWC, flows, objects, fields, permission sets, and more
- Use the `sfdx` project structure (Salesforce CLI creates this automatically)
- Add a `.gitignore` for `force.com` when creating the repository — GitHub suggests it automatically
- As a developer, your role: **develop in sandbox → commit to feature branch → raise PR**
- DevOps/pipeline handles deployment to higher sandboxes (UAT, staging, production)
- Never push directly to `master` — it represents production

> 📋 **Typical Salesforce branching flow:** `master (production)` → `uat` → `sit` → `developer feature branches`

---

## 15. Quick Command Reference

| Command | Purpose |
|---------|---------|
| `git --version` | Verify Git is installed |
| `git config --global user.name "Name"` | Set your display name for commits |
| `git config --global user.email "email"` | Set your email for commits |
| `git clone <url>` | Clone (download) a remote repository to your machine |
| `git status` | See which files have been changed |
| `git add <file>` OR `git add .` | Stage file(s) for committing |
| `git commit -m "message"` | Commit staged changes with a message |
| `git push` | Push local commits to the remote repository |
| `git pull` | Pull latest changes from the remote branch |
| `git checkout -b feature/my-feature` | Create and switch to a new branch |
| `git log` | View commit history with IDs |
| `git revert <commit-id>` | Safely undo a previous commit |

---

> *SwiftNotes | Version Control with Git & GitHub | Developer & Salesforce Edition*