# Copy this repo to BenVerkstedt/n8n-project-status-interviewer

Use this if the workflow needs to push to your personal GitHub (e.g. n8n token has no access to the Verkstedt org repo).

## 1. Create the new repo on GitHub

1. Go to [github.com/new](https://github.com/new).
2. **Owner:** Select **BenVerkstedt** (your account).
3. **Repository name:** `n8n-project-status-interviewer`.
4. **Public** (or Private if you prefer).
5. Do **not** add a README, .gitignore, or license (this repo already has them).
6. Click **Create repository**.

## 2. Push this code to the new repo

In this project folder, run:

```bash
# Add your new repo as a remote (e.g. named "benverkstedt")
git remote add benverkstedt https://github.com/BenVerkstedt/n8n-project-status-interviewer.git

# Push main (and all branches/tags if you want)
git push -u benverkstedt main
```

Optional: push all branches and tags:

```bash
git push benverkstedt --all
git push benverkstedt --tags
```

## 3. Use the new repo in n8n

In the **Project Status Consolidation** workflow, in the **Commit to GitHub** node:

- **Repository Owner:** `BenVerkstedt`
- **Repository Name:** `n8n-project-status-interviewer`
- **Branch:** `main` (or whatever the default branch is)
- **Credential:** A GitHub token for the BenVerkstedt account with `repo` scope.

Save and run the workflow again; it will push to your copy.

## 4. (Optional) Make the new repo the default remote

If you want this clone to track BenVerkstedt instead of verkstedt:

```bash
git remote rename origin origin-verkstedt
git remote rename benverkstedt origin
git branch -u origin/main main
```

Then `git push` and `git pull` will use BenVerkstedt/n8n-project-status-interviewer by default.
