# Git Workflow & Collaboration Guide

## 1. Branch Structure

We will use the following branches:

```text
main
└── development
    ├── feature/your-feature
    ├── feature/friend-feature
    └── fix/bug-name
```

### `main`

* Contains the stable version of the project.
* **Never work directly on `main`.**
* Only merge tested and completed work into `main`.

### `development`

* Main branch where completed features are combined and tested.
* Team members create feature branches from `development`.

### Feature branches

Each person works on their own branch.

Examples:

```text
feature/login-page
feature/dashboard
feature/photo-upload
fix/login-error
```

---

# 2. Before Starting Work

Always make sure your local repository is up to date.

Run:

```bash
git switch development
git pull origin development
```

Then create your own branch:

```bash
git switch -c feature/your-feature
```

Example:

```bash
git switch -c feature/login-page
```

---

# 3. While Working

Work only inside your own branch.

Check your current branch:

```bash
git branch
```

The branch with `*` is your current branch.

Example:

```text
  development
* feature/login-page
  main
```

You should **never see yourself working on `main`**.

---

# 4. Save Your Work

Check what changed:

```bash
git status
```

Add your files:

```bash
git add .
```

Commit your changes:

```bash
git commit -m "Add login page"
```

### Commit Message Format

Use clear messages.

Good:

```text
Add login page
Fix login validation
Update dashboard layout
Add photo upload functionality
Fix image processing error
Update README
```

Avoid:

```text
stuff
update
changes
asdf
final
final2
finalfinal
```

---

# 5. Push Your Branch

Push your branch to GitHub:

```bash
git push -u origin feature/your-feature
```

After the first push, you can usually use:

```bash
git push
```

---

# 6. Creating a Pull Request

When your feature is finished:

1. Push your branch to GitHub.
2. Open the repository on GitHub.
3. Create a **Pull Request**.
4. Set:

```text
base: development
compare: feature/your-feature
```

5. Explain what you changed.
6. Ask your friend to review it.
7. Fix any requested changes.
8. Merge into `development` only after both agree it is ready.

---

# 7. IMPORTANT: Do Not Merge Directly Into `main`

The normal process is:

```text
Feature Branch
      ↓
Pull Request
      ↓
development
      ↓
Testing
      ↓
Pull Request
      ↓
main
```

`main` should always contain a stable version.

---

# 8. Before Starting Another Feature

After your previous feature has been merged, update your local `development`:

```bash
git switch development
git pull origin development
```

Then create a new branch:

```bash
git switch -c feature/new-feature
```

Example:

```bash
git switch -c feature/profile-page
```

---

# 9. If Your Friend Updated `development`

Suppose your friend finished a feature and merged it into `development`.

Before starting your next task:

```bash
git switch development
git pull origin development
```

Then create your new branch:

```bash
git switch -c feature/new-feature
```

This makes sure your new work starts from the latest version.

---

# 10. If You Are Still Working on a Feature

If your feature is taking several days and your friend has added changes to `development`, update your branch carefully.

First commit your current work:

```bash
git add .
git commit -m "WIP: Continue login feature"
```

Then update `development`:

```bash
git switch development
git pull origin development
```

Go back to your branch:

```bash
git switch feature/your-feature
```

Then merge the latest development:

```bash
git merge development
```

If Git reports conflicts, **do not panic and do not randomly delete code**.

---

# 11. Handling Merge Conflicts

A conflict may look like:

```text
<<<<<<< HEAD
Your code
=======
Friend's code
>>>>>>> development
```

You need to decide which code should remain.

After fixing the conflict:

```bash
git add .
```

Then:

```bash
git commit -m "Resolve merge conflict"
```

Then continue working.

### IMPORTANT

If you don't understand a conflict, **ask your friend before choosing which code to delete.**

---

# 12. Pull Before Pushing

Before pushing, it is good practice to check whether your branch is behind.

```bash
git status
```

If you're working on a shared branch such as `development`, always pull first:

```bash
git pull origin development
```

Do not blindly force push.

### NEVER use:

```bash
git push --force
```

unless both teammates specifically agree and understand why it is necessary.

---

# 13. Don't Work on the Same Files at the Same Time When Possible

Try to divide the work.

Example:

### You

```text
src/
├── components/
│   ├── Login.jsx
│   └── Navbar.jsx
```

### Friend

```text
src/
├── components/
│   ├── Dashboard.jsx
│   └── Profile.jsx
```

This reduces merge conflicts.

However, if you both need to modify the same file, communicate before doing so.

---

# 14. Communicate Before Major Changes

Tell your teammate before:

* Changing the database structure
* Renaming important files
* Removing files
* Changing APIs
* Changing project architecture
* Installing/removing major dependencies
* Changing environment variables
* Modifying authentication
* Refactoring a large portion of the project

Example:

> "I'm going to change the user API structure. Don't work on that API until I'm finished."

---

# 15. Never Commit Secrets

Never commit:

```text
.env
API keys
passwords
database passwords
private tokens
credentials
```

Make sure `.gitignore` contains:

```gitignore
node_modules/
.env
.env.local
dist/
build/
```

If the project requires environment variables, create:

```text
.env.example
```

Example:

```env
API_URL=
DATABASE_URL=
API_KEY=
```

Do **not** put the real values in `.env.example`.

---

# 16. Recommended Daily Workflow

### Start of the day

```bash
git switch development
git pull origin development
```

Create your branch:

```bash
git switch -c feature/my-feature
```

---

### During development

```bash
git status
```

Make changes.

Then:

```bash
git add .
git commit -m "Describe what you changed"
```

Push:

```bash
git push
```

---

### Feature finished

Create a Pull Request:

```text
feature/my-feature
        ↓
   development
```

Ask your friend to review.

---

### After merging

```bash
git switch development
git pull origin development
```

Delete your local feature branch:

```bash
git branch -d feature/my-feature
```

Delete the remote branch if necessary:

```bash
git push origin --delete feature/my-feature
```

Then start the next feature.

---

# 17. Emergency: "I Messed Something Up"

### If you have uncommitted changes

First check:

```bash
git status
```

**Do not immediately run random Git commands.**

Ask your teammate before doing something destructive.

---

### If you accidentally committed something

Don't immediately delete or reset things.

Tell your teammate:

> "I accidentally committed something to the wrong branch. Don't pull/merge it yet."

Then figure out the safest solution together.

---

### If you accidentally worked on `main`

Stop.

Do not push.

Tell your teammate and fix the branch before continuing.

---

# 18. Golden Rules

### Rule 1

**Never work directly on `main`.**

### Rule 2

**Pull before starting new work.**

### Rule 3

**One feature = one branch.**

### Rule 4

**Commit small, meaningful changes.**

### Rule 5

**Communicate before major changes.**

### Rule 6

**Review Pull Requests before merging.**

### Rule 7

**Never commit `.env` or secrets.**

### Rule 8

**Never force push without agreement.**

### Rule 9

**Don't delete code you don't understand during a merge conflict.**

### Rule 10

**If you're unsure, stop and ask your teammate before running destructive Git commands.**

---

# 19. Our Basic Workflow

Remember this:

```text
                GitHub
                   │
                 main
                   │
             development
              /         \
             /           \
            ↓             ↓
       Your Branch    Friend's Branch
            │             │
            ↓             ↓
         commits        commits
            │             │
            └──────┬──────┘
                   ↓
             Pull Request
                   ↓
             development
                   ↓
                Testing
                   ↓
                 main
```

## Quick Version

Whenever you work:

```bash
git switch development
git pull origin development

git switch -c feature/my-feature

# Make changes

git add .
git commit -m "Describe changes"
git push -u origin feature/my-feature
```

Then:

```text
GitHub → Pull Request → Review → development → Test → main
```

**If something looks wrong, stop before pushing.**
