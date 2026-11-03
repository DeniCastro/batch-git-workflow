# batch-git-workflow
> Double-click Git workflow for classrooms: Windows batch scripts, one branch per student.


---

## What is this?

Teaching Git to a whole class on shared school computers is messy. Students forget to configure their name, mix up branches, overwrite each other's work, or get stuck on a command-line error before the lesson even starts.

This project solves that with **two small `.bat` files** that students double-click:

| Script | What it does |
|---|---|
| `start-class.bat` | Asks for a nickname, creates (or reopens) that student's own branch, and copies the latest lessons into it. |
| `end-class.bat` | Saves everything the student did, sends it to GitHub, and cleans the computer for the next student. |

The teacher keeps full visibility (every student's work is a branch with a clear history), and students never need a GitHub account, a password, or any Git configuration.

---

## How it works

```
 Student double-clicks                    Student works                    Student double-clicks
   start-class.bat          ──►     (VS Code, exercises, ...)     ──►       end-class.bat
          │                                                                       │
          ▼                                                                       ▼
  1. types a nickname                                                    1. saves (add + commit)
  2. gets their own branch                                               2. sends (push) to GitHub
  3. receives the latest lessons                                         3. goes back to main
                                                                         4. forgets the nickname
```

**The model, in one sentence:** every student works in a separate branch named after their nickname, so nobody can break anyone else's work and the teacher can read each student's history on GitHub.

---

## Features

- **One branch per student**, named after a nickname (letters, numbers and hyphens only).
- **Nicknames, not real names.** Commits are signed with the nickname and a fake email (`nickname@students.invalid`) that doesn't belong to anyone.
- **Input validation.** Spaces and special characters that break batch files are rejected, and nicknames are converted to lowercase.
- **Returning students are welcomed back.** If the nickname already exists, the script asks for confirmation and restores that student's work.
- **Always fresh lessons.** On every start, the lessons folder is copied from `main`, so new exercises appear without touching the computers.
- **Safety checks.** The scripts stop if the folder isn't clean, if the saved name doesn't match the branch, or if the upload fails (and never delete work in that case).
- **Clean computers.** After class, the machine returns to `main` with no identity configured.
- **Plain-English messages.** Every error tells the student to call the teacher, and the window never closes by itself.

---

## Requirements

- Windows 10 or 11
- [Git for Windows](https://git-scm.com/) (includes the credential manager)
- A GitHub repository for the class (**private** is recommended)
- A **fine-grained access token** restricted to that single repository

---

## Folder layout on each computer

```
C:\Classroom\
├── start-class.bat        ← students double-click this
├── end-class.bat          ← and this
└── class-repo\            ← the cloned class repository
    └── exercises\         ← the lessons (copied into each student's branch)
```

The scripts live **next to** the repository folder, not inside it. They find the repository using their own location, so the parent folder can be named anything.

---

## Setup (once per computer)

1. **Install Git for Windows** with the default options.
2. **Clone the class repository** into the parent folder:
   ```
   cd C:\Classroom
   git clone https://<account>@github.com/<account>/<class-repo>.git
   ```
   When the login window appears, choose the **token** option and paste the token as the password. Windows remembers it, and students never see it.
3. **Copy `start-class.bat` and `end-class.bat`** into `C:\Classroom`, next to the repository folder.
4. **Edit the two variables** near the top of `start-class.bat` (and the `REPO` line in `end-class.bat`) to match your names:
   ```bat
   set "REPO=%~dp0class-repo"
   set "LESSONS=exercises"
   ```
5. **Protect the scripts** so students can't edit them (Command Prompt as administrator):
   ```
   icacls C:\Classroom\start-class.bat /inheritance:r /grant:r "Administrators:F" "Users:RX"
   icacls C:\Classroom\end-class.bat /inheritance:r /grant:r "Administrators:F" "Users:RX"
   ```
6. If Git complains about *"dubious ownership"*, run once as the students' Windows user:
   ```
   git config --global --add safe.directory C:/Classroom/class-repo
   ```
7. **Test it:** run `start-class.bat` with a test nickname, create a file, run `end-class.bat`, and check the branch on GitHub.

> 💡 Use a standard (non-administrator) Windows account for students.

---

## The two-repository design

For safety, this setup uses two repositories:

| Repository | Visibility | Purpose |
|---|---|---|
| **This one** (the method) | Public | Scripts, documentation, and lesson material anyone can read. |
| **The class repository** | **Private** | Where students' branches live. Only the teacher and the class token can access it. |

Keep your lessons flowing in one direction: edit the original, then push it to the class repository. The scripts only ever **read** lessons from the class repository's `main` branch.

---

## Security notes

- **Token:** use a *fine-grained* personal access token, limited to the class repository only, with **Contents: Read and write** and an expiration date (one semester or less). Store it in a password manager. If it leaks or a computer is lost, delete it on GitHub right away and create a new one.
- **Never put the token in a file**, a script or this repository.
- **Don't sign in to your personal GitHub account** on the classroom computers.
- **Nicknames are an honor system.** Anyone who types another student's nickname and confirms will enter that branch. Keep a private table of *nickname → student*, and check commit times against class hours.
- **Branch protection:** GitHub rulesets are **not enforced on private repositories in free personal accounts**. Without them, anyone with the token could delete or overwrite branches from a terminal. Mitigate it with regular backups (`git clone --mirror`) and clear class rules, or use a plan that supports branch protection.
- **Public repositories show everything.** Never put answer keys, student data or credentials in a public repository.

---

## Common errors (and why they happen)

| Symptom | Likely cause | Fix |
|---|---|---|
| The file opens as text instead of running | The extension is really `.bat.txt` (hidden extensions) | Show file extensions in Explorer and rename it |
| Strange errors right at the start of a script | Wrong line endings or non-ASCII characters | Keep scripts ASCII-only with Windows (CRLF) line endings. A `.gitattributes` with `*.bat text eol=crlf` helps. |
| `Repository not found` | Not authenticated, or the wrong account's login is saved | Remove old `github.com` entries from Windows Credential Manager and sign in with the class account and token |
| `dubious ownership` | The folder was created by another Windows user | Run the `safe.directory` command from the setup |
| Upload fails | No internet, or the token expired | The work stays on the computer. Fix the cause and run `end-class.bat` again on the **same** computer. |
| `index.lock` or Git seems blocked | VS Code is open and running Git in the background | Close VS Code before `end-class.bat` |
| Nickname refused | Spaces, accents or symbols | Use only lowercase letters, numbers and hyphens |
| Old nickname fails after deleting its branch on GitHub | A stale local copy | `start-class.bat` runs `git fetch --prune` and restarts such branches from `main` automatically |

---

## A typical teacher routine

**Before class**
- Publish new lessons (push them to the class repository's `main`).
- Check that the computers have internet access.

**After class**
- Review the branches on GitHub.
- Update the backup mirror: `git remote update` inside your `--mirror` clone.

**Every semester**
- Renew the token before it expires.
- Delete old student branches and revoke the old token.

---

## Why batch files?

Batch scripts are the simplest automation Windows offers: a plain text file of commands that runs with a double-click, with nothing to install. They have quirks (special characters, line endings, and encoding), which is why these scripts validate input and avoid anything fancy. If you need more power, the same workflow translates well to PowerShell.

---

## Limitations

- Windows only.
- Students must have a working internet connection to start and end a class.
- Without paid branch protection, safety relies on backups and class rules.
- Nicknames are not authentication.

---

## Contributing

Ideas and improvements are welcome. Open an issue or a pull request. Please don't include any real names, school names, or credentials. 
Connect with me on [DeniCastro](https://github.com/DeniCastro).
