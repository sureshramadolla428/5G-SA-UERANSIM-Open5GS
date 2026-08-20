# Push this public 5G SA pack to GitHub

Git root: `C:\Users\sures\OneDrive\Desktop\3GPP_RAG_SA_LAB\github-ueransim-open5gs`

Do **not** git-init or push the parent folder `3GPP_RAG_SA_LAB` or `URRANSIM_Open5gs` (RAG, NTN, ATG, captures, career docs).

Suggested repo name: `5G-SA-UERANSIM-Open5GS`.

## Create and push (public)

```powershell
cd C:\Users\sures\OneDrive\Desktop\3GPP_RAG_SA_LAB\github-ueransim-open5gs
gh repo create 5G-SA-UERANSIM-Open5GS --public --source=. --remote=origin --push
```

Or:

```powershell
git remote add origin https://github.com/sureshramadolla428/5G-SA-UERANSIM-Open5GS.git
git branch -M main
git push -u origin main
```

## Do not

- `git push --force` to `main`
- Commit PCAPs, logs, 3GPP PDFs, NTN/ATG/RAG trees, or helper scripts (those are private)
- Add a project LICENSE that claims Open5GS or UERANSIM
- Add Cursor (or any tool) as a git co-author
