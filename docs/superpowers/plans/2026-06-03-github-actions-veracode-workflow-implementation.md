# GitHub Actions Veracode Multi-Module Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a GitHub Actions workflow that detects changed modules in PRs, packages them for Veracode scanning, and submits them to Veracode pipeline scan in parallel with configurable failure thresholds.

**Architecture:** Single workflow file with three sequential jobs: detect-changes (runs once, outputs module list), package-modules (matrix job over modules), veracode-scan (matrix job over modules, parallel submissions within each job).

**Tech Stack:** GitHub Actions, Git, Veracode CLI (installed via official install script), Veracode pipeline-scan-action v1.0.11

---

### Task 1: Create workflow skeleton with basic structure

**Files:**
- Create: `.github/workflows/veracode-multi-module-scan.yml`

- [ ] **Step 1: Create the workflows directory if it doesn't exist**

```bash
mkdir -p .github/workflows
```

- [ ] **Step 2: Create the workflow file with name, trigger, and job outline**

Create `.github/workflows/veracode-multi-module-scan.yml`:

```yaml
name: Veracode Multi-Module Scan

on:
  pull_request:
    branches: ['**']

jobs:
  detect-changes:
    name: Detect Changed Modules
    runs-on: ubuntu-latest
    outputs:
      modules: ${{ steps.detect.outputs.modules }}
    steps:
      # Will be populated in Task 2

  package-modules:
    name: Package Modules
    needs: detect-changes
    if: needs.detect-changes.outputs.modules != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: ${{ fromJson(needs.detect-changes.outputs.modules) }}
    steps:
      # Will be populated in Task 3

  veracode-scan:
    name: Veracode Pipeline Scan
    needs: package-modules
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: ${{ fromJson(needs.detect-changes.outputs.modules) }}
    steps:
      # Will be populated in Task 4
```

- [ ] **Step 3: Commit the skeleton**

```bash
git add .github/workflows/veracode-multi-module-scan.yml
git commit -m "scaff: add GitHub Actions workflow skeleton for Veracode scanning"
```

---

### Task 2: Implement detect-changes job

**Files:**
- Modify: `.github/workflows/veracode-multi-module-scan.yml` (detect-changes steps)

- [ ] **Step 1: Implement checkout and fetch base branch**

Replace the `# Will be populated in Task 2` comment in `detect-changes.steps` with:

```yaml
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Fetch base branch
        run: |
          git fetch origin ${{ github.base_ref }}:refs/remotes/origin/${{ github.base_ref }} 2>/dev/null || true
```

- [ ] **Step 2: Implement git diff to detect changed modules**

Add this step after the fetch step:

```yaml
      - name: Detect changed modules
        id: detect
        run: |
          # Get all changed files from base to PR branch
          CHANGED_FILES=$(git diff --name-only origin/${{ github.base_ref }}...HEAD 2>/dev/null || git diff --name-only HEAD~1...HEAD)
          
          # Extract unique top-level folder names
          MODULES=$(echo "$CHANGED_FILES" | cut -d'/' -f1 | sort -u | grep -v '^$')
          
          # Filter to only folders containing .csproj files
          FILTERED_MODULES=()
          for MODULE in $MODULES; do
            if [ -f "$MODULE"/*.csproj ]; then
              FILTERED_MODULES+=("$MODULE")
            fi
          done
          
          # Convert to JSON array
          if [ ${#FILTERED_MODULES[@]} -eq 0 ]; then
            MODULES_JSON="[]"
          else
            MODULES_JSON=$(printf '%s\n' "${FILTERED_MODULES[@]}" | jq -R . | jq -s .)
          fi
          
          echo "modules=$MODULES_JSON" >> $GITHUB_OUTPUT
          echo "Changed modules: $MODULES_JSON"
```

- [ ] **Step 3: Verify the output and commit**

```bash
git add .github/workflows/veracode-multi-module-scan.yml
git commit -m "feat: implement detect-changes job with git diff module detection"
```

---

### Task 3: Implement package-modules job

**Files:**
- Modify: `.github/workflows/veracode-multi-module-scan.yml` (package-modules steps)

- [ ] **Step 1: Implement checkout for package-modules job**

Replace the `# Will be populated in Task 3` comment in `package-modules.steps` with:

```yaml
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
```

- [ ] **Step 2: Add Veracode CLI installation**

Add this step after checkout:

```yaml
      - name: Install Veracode CLI
        run: |
          curl -fsS https://tools.veracode.com/veracode-cli/install | sh
          ./veracode version
```

- [ ] **Step 3: Implement veracode package command**

Add this step after CLI installation:

```yaml
      - name: Package module with Veracode
        run: |
          ./veracode package \
            -s "${{ matrix.module }}" \
            -a \
            -o "veraout/${{ matrix.module }}" \
            -t directory
```

- [ ] **Step 4: Upload artifacts for each module**

Add this step after packaging:

```yaml
      - name: Upload packaged module artifacts
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: veracode-${{ matrix.module }}
          path: veraout/${{ matrix.module }}/
          retention-days: 1
```

- [ ] **Step 5: Commit the package job**

```bash
git add .github/workflows/veracode-multi-module-scan.yml
git commit -m "feat: implement package-modules job with Veracode CLI packaging"
```

---

### Task 4: Implement veracode-scan job

**Files:**
- Modify: `.github/workflows/veracode-multi-module-scan.yml` (veracode-scan steps)

- [ ] **Step 1: Implement checkout and download artifacts**

Replace the `# Will be populated in Task 4` comment in `veracode-scan.steps` with:

```yaml
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Download packaged artifacts
        uses: actions/download-artifact@v4
        with:
          name: veracode-${{ matrix.module }}
          path: veraout/${{ matrix.module }}/
```

- [ ] **Step 2: List downloaded files for debugging**

Add this step after download:

```yaml
      - name: List downloaded artifacts
        run: |
          echo "Artifacts for ${{ matrix.module }}:"
          find veraout/${{ matrix.module }}/ -type f \( -name "*.zip" -o -name "*.tar.gz" \)
```

- [ ] **Step 3: Install Veracode CLI**

Add this step after listing:

```yaml
      - name: Install Veracode CLI
        run: |
          curl -fsS https://tools.veracode.com/veracode-cli/install | sh
```

- [ ] **Step 4: Submit all packages to Veracode pipeline scan in parallel**

Add this step after CLI installation:

```yaml
      - name: Submit packages to Veracode pipeline scan
        run: |
          ZIPS=$(find veraout/${{ matrix.module }}/ -type f \( -name "*.zip" -o -name "*.tar.gz" \))
          
          if [ -z "$ZIPS" ]; then
            echo "No scan packages found for ${{ matrix.module }}"
            exit 1
          fi
          
          # Submit all ZIPs in parallel
          PIDS=()
          for ZIP in $ZIPS; do
            echo "Submitting $ZIP to Veracode..."
            ./veracode scan \
              --vid "${{ secrets.VERACODE_API_ID }}" \
              --vkey "${{ secrets.VERACODE_API_KEY }}" \
              --file "$ZIP" \
              --fail-on-severity "critical" \
              --fail-build true &
            PIDS+=($!)
          done
          
          # Wait for all submissions and collect exit codes
          FAILED=0
          for PID in "${PIDS[@]}"; do
            if ! wait $PID; then
              FAILED=1
            fi
          done
          
          if [ $FAILED -eq 1 ]; then
            echo "One or more scans failed"
            exit 1
          fi
```

- [ ] **Step 5: Commit the scan job**

```bash
git add .github/workflows/veracode-multi-module-scan.yml
git commit -m "feat: implement veracode-scan job with parallel pipeline submissions"
```

---

### Task 5: Validate workflow syntax and test

**Files:**
- Verify: `.github/workflows/veracode-multi-module-scan.yml`

- [ ] **Step 1: Validate workflow syntax**

```bash
# Check YAML syntax
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/veracode-multi-module-scan.yml'))" && echo "YAML is valid"
```

Expected: Output "YAML is valid" with no errors.

- [ ] **Step 2: Verify secrets are referenced correctly**

```bash
grep -c "secrets.VERACODE_API_ID" .github/workflows/veracode-multi-module-scan.yml
grep -c "secrets.VERACODE_API_KEY" .github/workflows/veracode-multi-module-scan.yml
```

Expected: Each appears at least once in the veracode-scan job.

- [ ] **Step 3: Verify job dependencies and outputs**

```bash
grep -E "^  [a-z]" .github/workflows/veracode-multi-module-scan.yml
grep "needs:" .github/workflows/veracode-multi-module-scan.yml
grep "outputs:" .github/workflows/veracode-multi-module-scan.yml
```

Expected: 
- Three jobs defined (detect-changes, package-modules, veracode-scan)
- `package-modules` depends on `detect-changes`
- `veracode-scan` depends on `package-modules`
- `detect-changes` has outputs section with `modules`

- [ ] **Step 4: Verify matrix configuration**

```bash
grep -A1 "strategy:" .github/workflows/veracode-multi-module-scan.yml
```

Expected: Both `package-modules` and `veracode-scan` have matrix.module configuration.

- [ ] **Step 5: Final commit**

```bash
git add .github/workflows/veracode-multi-module-scan.yml
git commit -m "test: validate workflow syntax and structure"
```

---

## Self-Review Against Spec

**Spec coverage check:**
- ✅ PR trigger: Implemented in `on.pull_request`
- ✅ detect-changes job: Diffs base vs PR branch, extracts modules, outputs JSON array
- ✅ package-modules job: Matrix over modules, installs Veracode CLI via install script, runs veracode package, uploads artifacts
- ✅ veracode-scan job: Matrix over modules, downloads artifacts, installs Veracode CLI, submits to pipeline scan in parallel
- ✅ Multiple ZIPs per module: Artifact upload captures all files; scan step loops through all ZIPs found
- ✅ Parallel submission: `&` background processes with PID collection and wait loop
- ✅ Fail on scan results: `--fail-on-severity` and `--fail-build` flags; exit code handling
- ✅ No PR annotations: No summary or check step outputs generated

**No placeholders:** All steps contain actual code and commands with Veracode CLI paths. ✅

**Type/naming consistency:** 
- Module matrix variable: `${{ matrix.module }}` used consistently
- JSON output: `modules` output used in `fromJson()` consistently
- Secrets: `secrets.VERACODE_API_ID` and `secrets.VERACODE_API_KEY` used consistently
- CLI path: `./veracode` used consistently (installs to current directory)

---

Plan saved to `docs/superpowers/plans/2026-06-03-github-actions-veracode-workflow-implementation.md`.

Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
