### Stage 1: Repo Setup & Branch Preparation (`/stage1-setup`)

This stage gets the repo ready: clone, add upstream remote, and create the `dev` and `base` branches
needed for the merge. You can also run sub-steps individually: `/stage1-clone`, `/stage1-create-dev`,
`/stage1-create-base`.

#### Step 1: Clone the fork

```bash
git clone https://github.com/flagos-ai/TransformerEngine-FL.git
cd TransformerEngine-FL
```

Skip if already cloned — run the Repo Detection Preamble above instead.

#### Step 2: Add upstream remote and fetch

```bash
git remote -v | grep upstream
# If not present:
git remote add upstream https://github.com/Nvidia/TransformerEngine.git
git fetch upstream --tags
```

#### Step 3: Create dev branch from upstream release (`/stage1-create-dev`)

The `dev` branch mirrors the target upstream release exactly — no fork-specific changes.

1. Identify the target release branch/tag. Default is `release_v2.14`, but ask the user to confirm:
   ```bash
   git branch -r | grep upstream/release
   ```

2. Create the dev branch:
   ```bash
   git checkout -b dev upstream/release_v2.14
   ```

3. Record the sync point — create `SYNC_POINT.md` at repo root:
   ```markdown
   # Upstream Sync Point
   - Upstream: Nvidia/TransformerEngine
   - Branch: release_v2.14
   - Commit SHA: <output of `git rev-parse HEAD`>
   - Sync Date: <current date>
   - Synced By: <user>
   ```

4. Verify:
   ```bash
   git log --oneline -5
   git diff upstream/release_v2.14
   ```
   The diff must be empty (only SYNC_POINT.md should differ if committed).

#### Step 4: Create base branch from fork's original upstream (`/stage1-create-base`)

The `base` branch represents the upstream version the fork was originally based on. This is needed
for accurate three-way merges.

```bash
git fetch upstream release_v2.9
git checkout -b base upstream/release_v2.9
git log --oneline -5
git diff upstream/release_v2.9
```
The diff must be empty.

**Success criteria:** `dev` and `base` branches exist and match their respective upstream releases
commit-for-commit.

---

### Stage 2: Identify Plugin Changes (`/stage2-diff-plugin-changes`)

Before merging, you need to understand exactly what the fork added on top of upstream. This diff
between `base` (upstream release_v2.9) and `main` (fork) reveals all plugin-related changes — the
files you must protect during the merge.

**Steps:**

1. Run the Repo Detection Preamble to ensure you are in the TransformerEngine-FL directory.

2. Generate a summary of all changes the fork introduced:
   ```bash
   git diff base..main --stat
   ```

3. Generate the full diff and save it for reference:
   ```bash
   git diff base..main > plugin_changes.diff
   ```

4. Identify plugin-specific changes:
   ```bash
   # Files added or modified in plugin directory
   git diff base..main --name-status -- 'transformer_engine/plugin/'

   # CUDA patches added or modified
   git diff base..main --name-status -- 'transformer_engine/__init__.py'

   # Build system changes for plugin support
   git diff base..main -- setup.py CMakeLists.txt pyproject.toml

   # API changes (e.g. torch.Tensor('cuda') -> torch.Tensor('TE_DEVICE_TYPE'))
   # Detailed changes captured by diff base..main
   git diff base..main --name-status -- 'transformer_engine/pytorch/'
   ```

5. Record the changes — save a structured summary to `PLUGIN_CHANGES.md`:
   ```markdown
   # Plugin Changes (base → main)

   ## New Files (added by fork)
   <list of files only in main, not in base>

   ## Modified Files (changed by fork)
   <list of files that exist in both but differ>

   ## Plugin Directory Contents
   <full listing of transformer_engine/common/plugin/>

   ## CUDA Patches Contents
   <full listing of transformer_engine/common/cuda_patches/>

   ## Build System Modifications
   <summary of plugin-related changes in setup.py, CMakeLists.txt, pyproject.toml>

   ## Python Binding Modifications
   <summary of plugin-related changes in transformer_engine/pytorch/>
   ```

This record is critical — during Stage 3 (Merge & Conflict Resolution), use it to verify that every
plugin change from main survives the merge. If a file listed here has a conflict, it needs
careful attention.

**Success criteria:** `plugin_changes.diff` and `PLUGIN_CHANGES.md` generated, all fork-specific
changes catalogued.

