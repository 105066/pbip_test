# Git & GitHub Guide for This Project

A first-timer's manual for saving and syncing this Power BI Project (PBIP) with GitHub,
based on the setup we did on 2026-09-16.

## 1. One-time machine setup

### Install Git
Download and run the installer from https://git-scm.com/download/win — default options are fine.

> If `winget` is broken on your machine (source cache error), just use the installer above instead
> of fighting with `winget source reset` (which needs admin rights).

### Install GitHub CLI (`gh`) — portable, no admin required
```powershell
# 1. Find the latest Windows release
$release = Invoke-RestMethod -Uri "https://api.github.com/repos/cli/cli/releases/latest" -Headers @{ "User-Agent" = "setup" }
$asset = $release.assets | Where-Object { $_.name -match "windows_amd64\.zip$" }

# 2. Download and extract it to a per-user folder
$dest = "$env:LOCALAPPDATA\GitHubCLI"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
$zip = "$env:TEMP\gh_cli.zip"
Invoke-WebRequest -Uri $asset.browser_download_url -OutFile $zip
Expand-Archive -Path $zip -DestinationPath $dest -Force
Remove-Item $zip

# 3. Add it to your PATH permanently (per-user, no admin needed)
$ghBin = Get-ChildItem -Recurse -Filter "gh.exe" -Path $dest | Select-Object -First 1 -ExpandProperty DirectoryName
$userPath = [System.Environment]::GetEnvironmentVariable("Path","User")
[System.Environment]::SetEnvironmentVariable("Path", "$userPath;$ghBin", "User")
```
Close and reopen your terminal afterward so the new PATH takes effect.

### Configure your Git identity (once per machine)
```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### Log in to GitHub
```powershell
gh auth login
```
Choose: **GitHub.com** → **HTTPS** → **Yes, authenticate Git** → **Login with a web browser**.
It gives you a one-time code and opens your browser — paste the code and approve.

## 2. Core concepts explained

You don't need to know how Git works internally to use it, but these four ideas explain
almost everything you'll run into.

### Commits
A **commit** is a saved snapshot of your entire project at a point in time, plus a message
describing what changed. Commits are the "undo points" of your project — you can always go
back to any previous commit. Each commit has a unique ID (a hash like `a527f4e`) and points to
the commit that came before it, forming a chain (history):

```
9fb8a07 ── a527f4e ── (next commit)
"Initial commit"   "Add Certificate Display column"
```

Nothing is saved permanently until you `git commit`. Editing a file only changes your
working copy; `git add` stages it (marks it ready); `git commit` locks it into history.

### Diffs
A **diff** is the line-by-line difference between two versions of a file (e.g., your edits vs.
the last commit, or one commit vs. another). This is what makes Git useful for reviewing
changes — instead of re-reading a whole `.tmdl` file, you see exactly what was added/removed:

```diff
	column Product_Display
		dataType: string
+	column 'Certificate Display' = Data[certificate_name] & " (Rev " & Data[rev_number] & ")"
+		dataType: string
```
- `git diff` → changes not yet staged
- `git diff --staged` → changes staged and about to be committed
- `git diff <commit1> <commit2>` → changes between two commits
- GitHub also shows diffs visually on the commit and pull-request pages (lines removed in red,
  added in green).

### Branches
A **branch** is an independent line of development — a movable pointer to a commit. Every repo
starts with one default branch (here, `master`). Branches let you try something (e.g., a new
measure or a redesigned page) without touching the working version:

```
master:   9fb8a07 ── a527f4e ─────────────────┐
                                                 ├── (still separate until merged)
feature:                          └── 3f1a2b9 ──┘
```

Typical flow:
```powershell
git checkout -b feature/new-measure   # create + switch to a new branch
# ...edit files, commit as usual...
git push -u origin feature/new-measure  # publish the branch to GitHub
```
Then open a pull request (`gh pr create`) to review and merge it into `master`.

### Merges
A **merge** brings the changes from one branch into another. When you merge a feature branch
back into `master`, Git combines both histories:

```
master:   9fb8a07 ── a527f4e ────────────────●  (merge commit)
                                             /
feature:                    └── 3f1a2b9 ────┘
```
- If the changes don't overlap, Git merges automatically.
- If the same lines were changed differently on both branches, you get a **merge conflict** —
  Git marks the conflicting lines in the file with `<<<<<<<`, `=======`, `>>>>>>>` and asks you
  to manually choose/edit the final result, then `git add` the file and `git commit` to finish
  the merge.
- On GitHub, merging a pull request does this for you with a button (Merge / Squash / Rebase).

### Putting it together
1. You edit files → `git diff` to review → `git add` to stage → `git commit` to save a snapshot.
2. Work on risky/experimental changes on a **branch** so `master` stays stable.
3. `git push` sends your commits to GitHub; `git pull` brings down others' commits.
4. When a branch is ready, **merge** it back (directly, or via a pull request on GitHub).

## 3. Setting up this repo (already done, for reference)

```powershell
git init
git add .
git commit -m "Initial commit: pbip_test Power BI project"
gh repo create pbip_test --public --source=. --remote=origin --push
```

`gh repo create ... --source=. --remote=origin --push` does three things at once:
creates the GitHub repo, wires it up as `origin`, and pushes your first commit.

## 4. `.gitignore` for PBIP projects

Power BI Desktop writes local-only cache/settings files that shouldn't be committed
(they're machine-specific and can be large). Our `.gitignore`:

```
**/.pbi/localSettings.json
**/.pbi/cache.abf
**/*.pbix
**/.vs/
**/bin/
**/obj/
.DS_Store
Thumbs.db
```

## 5. Everyday workflow — the 3 commands you'll use most

Whenever you change something in Power BI Desktop (or edit `.tmdl`/`.json` files directly):

```powershell
git add -A                     # stage all changes
git commit -m "Describe what changed"
git push                       # send it to GitHub
```

To get other people's changes (or your own from another machine):
```powershell
git pull
```

## 6. Handy Git commands cheat sheet

| Command | What it does |
|---|---|
| `git status` | Shows what's changed/staged/untracked |
| `git diff` | Shows line-by-line changes not yet staged |
| `git diff --staged` | Shows changes staged for the next commit |
| `git add <file>` | Stage a specific file |
| `git add -A` | Stage everything (new, modified, deleted) |
| `git commit -m "message"` | Save staged changes as a new commit |
| `git log --oneline` | Compact history of commits |
| `git push` | Upload commits to GitHub |
| `git pull` | Download and merge commits from GitHub |
| `git branch` | List branches |
| `git checkout -b <name>` | Create and switch to a new branch |
| `git checkout <name>` | Switch to an existing branch |
| `git restore <file>` | Discard uncommitted changes to a file |
| `git restore --staged <file>` | Unstage a file (keep the edits) |
| `git clone <url>` | Copy a GitHub repo to your machine |
| `git remote -v` | Show configured remotes (e.g. `origin`) |

## 7. Handy GitHub CLI (`gh`) cheat sheet

| Command | What it does |
|---|---|
| `gh auth login` | Log in to GitHub |
| `gh auth status` | Check who you're logged in as |
| `gh repo create <name> --public/--private --source=. --push` | Create a repo from the current folder |
| `gh repo view --web` | Open the current repo in the browser |
| `gh repo clone <owner>/<repo>` | Clone a repo |
| `gh pr create` | Open a pull request from your current branch |
| `gh pr view --web` | View the current PR in the browser |
| `gh issue create` | Create an issue |
| `gh browse` | Open the current repo/file in the browser |

## 7. Tips & gotchas

- **PATH not updating in an open terminal**: after installing something new, existing terminal
  sessions won't see it. Open a new terminal, or refresh manually:
  ```powershell
  $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine") + ";" + [System.Environment]::GetEnvironmentVariable("Path","User")
  ```
- **Commit messages**: keep them short and descriptive, e.g. `"Add Certificate Display calculated column"`
  rather than `"update"`. Future-you will thank you when scanning `git log`.
- **Don't commit secrets**: never commit connection strings, passwords, or API keys. If you
  accidentally do, treat the credential as compromised and rotate it — removing it from a later
  commit does not remove it from history.
- **OneDrive + Git**: this project lives inside a OneDrive-synced folder. That's fine, but avoid
  editing the same file in Power BI Desktop on two machines at the same time — OneDrive sync and
  Git can both try to "resolve" conflicts and confuse each other. Prefer `git pull` before you
  start working, and `git push` when you're done.
- **PBIP file structure**: `.Report` and `.SemanticModel` folders are plain text/JSON/TMDL, which
  is exactly why they diff and merge nicely in Git (unlike a single binary `.pbix` file).
- **Large/binary files**: if you ever need to commit a `.pbix` or images, consider
  [Git LFS](https://git-lfs.com/) so the repo doesn't bloat.
- **Undo a bad commit that's already pushed**: prefer `git revert <commit>` (creates a new commit
  undoing the change) over `git reset` on shared history, since it doesn't rewrite history other
  people may have already pulled.

## 8. Where this repo lives

- Remote: https://github.com/105066/pbip_test
- Default branch: `master`
