# Overleaf Sync Setup Plan

## Overview

This document outlines the plan to automatically sync the generated LaTeX output from the main MSA repository to a separate repository that Overleaf can sync with.

## Architecture

```
MSA Repository (Markdown Source)
    ↓ (you edit and push)
GitHub Actions builds manuscript
    ↓ (generates manuscript.tex)
MSA-latex Repository (LaTeX Output Only)
    ↓ (Overleaf syncs)
Overleaf Project
```

## Repositories

### 1. Main Repository: `bnshin83/MSA`
- **Purpose**: Source of truth for manuscript content
- **Contents**: Markdown files, images, build scripts
- **Workflow**: Authors edit markdown, push to GitHub
- **Branch**: `main`

### 2. LaTeX Repository: `bnshin83/MSA-latex` (NEW)
- **Purpose**: LaTeX output for Overleaf integration
- **Contents**: `manuscript.tex`, images, bibliography files
- **Workflow**: Automatically updated by GitHub Actions
- **Branch**: `main`

## Implementation Steps

### Phase 1: GitHub Setup (15 minutes)

#### Step 1.1: Create New Repository
1. Go to https://github.com/new
2. Repository name: `MSA-latex`
3. Description: "LaTeX output for MSA manuscript (auto-synced from bnshin83/MSA)"
4. Visibility: Choose Public or Private (your preference)
5. **IMPORTANT**: Do NOT initialize with README, .gitignore, or license
6. Click "Create repository"

#### Step 1.2: Generate Personal Access Token (PAT)
1. Go to https://github.com/settings/tokens
2. Click "Generate new token (classic)"
3. Token name: `MSA-latex-sync`
4. Expiration: Choose appropriate duration (90 days, 1 year, or no expiration)
5. Select scopes:
   - ✅ `repo` (full control of private repositories)
6. Click "Generate token"
7. **COPY THE TOKEN** - you won't see it again!
8. Save it temporarily in a secure location

#### Step 1.3: Add Token as Repository Secret
1. Go to https://github.com/bnshin83/MSA/settings/secrets/actions
2. Click "New repository secret"
3. Name: `LATEX_SYNC_TOKEN`
4. Secret: Paste the token from Step 1.2
5. Click "Add secret"

### Phase 2: GitHub Actions Modification (10 minutes)

#### Step 2.1: Update Workflow File
Edit `.github/workflows/manubot.yaml`:

**Location**: After line 98 (after "Upload Artifacts" step)

**Insert this new step**:

```yaml
      - name: Sync LaTeX to separate repo for Overleaf
        if: github.ref == env.DEFAULT_BRANCH_REF && github.event_name != 'pull_request' && !github.event.repository.fork
        env:
          LATEX_SYNC_TOKEN: ${{ secrets.LATEX_SYNC_TOKEN }}
        run: |
          # Clone the latex-only repository
          git clone https://x-access-token:${LATEX_SYNC_TOKEN}@github.com/bnshin83/MSA-latex.git latex-repo
          cd latex-repo

          # Configure git
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          # Copy generated LaTeX and supporting files
          cp ../output/manuscript.tex .
          cp ../output/references.json . 2>/dev/null || true

          # Copy images if they exist
          if [ -d ../content/images ]; then
            mkdir -p images
            cp -r ../content/images/* images/ 2>/dev/null || true
          fi

          # Create a README for context
          cat > README.md << 'EOF'
          # MSA Manuscript - LaTeX Output

          This repository contains the auto-generated LaTeX output from the main MSA repository.

          **DO NOT EDIT FILES DIRECTLY IN THIS REPOSITORY**

          - Source repository: https://github.com/bnshin83/MSA
          - This repository is automatically updated when changes are pushed to the main repository
          - Use this repository to sync with Overleaf

          ## Files

          - `manuscript.tex` - Main LaTeX manuscript file
          - `images/` - Figures and images
          - `references.json` - Citation metadata

          Last updated: $(date -u)
          EOF

          # Commit and push changes
          git add .
          if ! git diff --quiet --cached; then
            git commit -m "Auto-sync from MSA repo - $(date -u '+%Y-%m-%d %H:%M:%S UTC')"
            git push
          else
            echo "No changes to commit"
          fi
```

#### Step 2.2: Commit and Push Workflow Changes
```bash
git add .github/workflows/manubot.yaml
git commit -m "Add automatic LaTeX sync to MSA-latex repository"
git push origin main
```

### Phase 3: Overleaf Integration (5 minutes)

#### Step 3.1: Link Overleaf to MSA-latex Repository
1. Go to https://www.overleaf.com/project
2. Click "New Project" → "Import from GitHub"
3. Select repository: `bnshin83/MSA-latex`
4. Click "Import to Overleaf"

#### Step 3.2: Configure Overleaf Project
1. Set main document to `manuscript.tex`
2. Enable "Auto-compile"
3. Test compilation

### Phase 4: Testing (5 minutes)

#### Step 4.1: Test the Sync
1. Make a small change to any markdown file in MSA repository:
   ```bash
   # Example: Add a test line to abstract
   echo "Test sync." >> content/01.abstract.md
   git add content/01.abstract.md
   git commit -m "Test: verify auto-sync to MSA-latex"
   git push origin main
   ```

2. Monitor GitHub Actions:
   - Go to https://github.com/bnshin83/MSA/actions
   - Watch the workflow run
   - Verify "Sync LaTeX to separate repo for Overleaf" step succeeds

3. Check MSA-latex repository:
   - Go to https://github.com/bnshin83/MSA-latex
   - Verify `manuscript.tex` was updated
   - Check commit timestamp

4. Check Overleaf:
   - Open your Overleaf project
   - Click "GitHub" → "Pull GitHub changes to Overleaf"
   - Verify the test change appears

#### Step 4.2: Cleanup Test
```bash
# Remove test line from abstract
git checkout content/01.abstract.md
git commit -m "Remove test line"
git push origin main
```

## Workflow After Setup

### For Regular Writing Sessions

```bash
# 1. Write in markdown files
vim content/02.introduction.md

# 2. Build locally to preview (optional)
conda activate manubot
bash build/build.sh
open output/manuscript.html

# 3. Commit and push
git add content/*.md
git commit -m "Update introduction section"
git push origin main

# 4. Wait 2-5 minutes for GitHub Actions to complete

# 5. In Overleaf: Menu → GitHub → Pull GitHub changes
```

### For Collaborators

**Markdown editors** (most collaborators):
- Clone `bnshin83/MSA`
- Edit markdown files
- Push to main branch
- Changes automatically appear in Overleaf

**LaTeX-only editors** (if needed):
- Work directly in Overleaf
- **WARNING**: Changes in Overleaf will be overwritten on next sync
- Use Overleaf only for final formatting before submission

## Important Notes

### Sync Behavior
- ✅ One-way sync: MSA → MSA-latex → Overleaf
- ⚠️ Changes in Overleaf are NOT synced back to MSA
- ⚠️ Each push to MSA overwrites MSA-latex content

### When to Edit in Overleaf
- Final formatting adjustments before submission
- Journal-specific LaTeX requirements
- Last-minute edits when markdown is frozen

### When to Edit in Markdown (MSA repo)
- All active writing
- Content changes
- Adding citations
- Structural changes
- **Adding figures**: Upload to `content/images/` first, then reference in Markdown

### ⚠️ CRITICAL: Adding Figures
**DO NOT** add figures directly in Overleaf during active writing!

**Correct workflow for adding figures:**
1. Add figure file to `content/images/` in MSA repo
2. Commit and push to MSA repo
3. Reference in Markdown: `![Caption](images/figure.png)`
4. GitHub Actions will sync image to Overleaf automatically

**Why:** Any push to MSA repo will overwrite the MSA-latex repository, deleting any figures added directly in Overleaf.

## Troubleshooting

### Sync Not Working
1. Check GitHub Actions logs: https://github.com/bnshin83/MSA/actions
2. Verify `LATEX_SYNC_TOKEN` secret is set correctly
3. Ensure MSA-latex repository exists and is accessible

### Overleaf Not Updating
1. Manually pull: Menu → GitHub → Pull GitHub changes to Overleaf
2. Check MSA-latex repository was updated
3. Verify Overleaf is linked to correct repository

### Merge Conflicts
- Should not occur with one-way sync
- If they do: Overleaf changes may conflict with auto-sync
- Resolution: Choose to keep GitHub version (MSA-latex)

## Security Considerations

### Personal Access Token
- Token has full repository access
- Store securely in GitHub Secrets (done in Step 1.3)
- Never commit token to repository
- Set appropriate expiration date
- Regenerate if compromised

### Repository Visibility
- MSA: Can be public or private
- MSA-latex: Should match MSA visibility
- Overleaf: Can be private regardless of GitHub repo visibility

## Future Enhancements

### Possible Additions
- [ ] Sync bibliography file (`.bib`) if generated
- [ ] Include supplementary materials
- [ ] Add version tags/releases
- [ ] Notification on sync completion
- [ ] Bi-directional sync (complex, not recommended initially)

## Timeline Summary

| Phase | Duration | Description |
|-------|----------|-------------|
| Phase 1 | 15 min | Create repo, generate token, add secret |
| Phase 2 | 10 min | Modify GitHub Actions workflow |
| Phase 3 | 5 min | Link Overleaf |
| Phase 4 | 5 min | Test the complete pipeline |
| **Total** | **35 min** | **Complete setup** |

## Checklist

### Pre-Setup
- [ ] Have admin access to bnshin83/MSA repository
- [ ] Have GitHub account with ability to create repositories
- [ ] Have Overleaf account linked to GitHub

### Setup Checklist
- [ ] Created MSA-latex repository on GitHub
- [ ] Generated Personal Access Token
- [ ] Added LATEX_SYNC_TOKEN secret to MSA repository
- [ ] Modified .github/workflows/manubot.yaml
- [ ] Committed and pushed workflow changes
- [ ] Linked Overleaf to MSA-latex repository
- [ ] Tested sync by making a test commit
- [ ] Verified LaTeX appears in Overleaf

### Post-Setup
- [ ] Documented workflow for collaborators
- [ ] Cleaned up test commits
- [ ] Shared Overleaf project with team (if needed)

## Support

If you encounter issues:
1. Check GitHub Actions logs for error messages
2. Verify all secrets are correctly set
3. Ensure repository permissions are correct
4. Review this document for missed steps

## Completion

Once all checklist items are complete, you will have:
- ✅ Automatic LaTeX generation on every push
- ✅ Separate repository for clean Overleaf integration
- ✅ One-way sync from markdown to LaTeX
- ✅ Ability to preview in Overleaf immediately after pushing

---

**Created**: 2025-10-31
**Repository**: bnshin83/MSA
**Purpose**: Enable seamless Overleaf integration for collaborative manuscript editing
