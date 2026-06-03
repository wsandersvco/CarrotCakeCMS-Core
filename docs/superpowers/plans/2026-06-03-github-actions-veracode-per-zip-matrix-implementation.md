# GitHub Actions Veracode Per-ZIP Matrix Workflow Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor the Veracode scanning workflow to matrix over individual ZIP files instead of modules, enabling true parallel scanning with granular failure reporting.

**Architecture:** Three-job pipeline where `package-modules` outputs ZIP file lists per module, `veracode-scan` combines those outputs into a single flat array, then matrices over individual ZIPs to scan each in parallel using the official pipeline-scan-action.

**Tech Stack:** GitHub Actions, Git, Veracode CLI, veracode/Veracode-pipeline-scan-action@v1.0.11

---

### Task 1: Add ZIP file list output to package-modules job

**Files:**
- Modify: `.github/workflows/veracode-multi-module-scan.yml` (package-modules steps, add output step after packaging)

- [ ] **Step 1: Read the current package-modules job**

Read the workflow file to see the current package step structure.

Run: `grep -A 30 "name: Package module with Veracode" .github/workflows/veracode-multi-module-scan.yml`

Expected: Output shows the packaging step and artifact upload.

- [ ] **Step 2: Add ZIP file list step after packaging**

After the "Package module with Veracode" step (around line 100-107), insert a new step to find all generated ZIPs and output them:

Replace this section (lines 100-115):
```yaml
      - name: Package module with Veracode
        run: |
          set -e
          ./veracode package \
            -s "${{ matrix.module }}" \
            -a \
            -o "veraout/${{ matrix.module }}" \
            -t directory

      - name: Upload packaged module artifacts
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: veracode-${{ matrix.module }}
          path: veraout/${{ matrix.module }}/
          retention-days: 1
```

With:
```yaml
      - name: Package module with Veracode
        run: |
          set -e
          ./veracode package \
            -s "${{ matrix.module }}" \
            -a \
            -o "veraout/${{ matrix.module }}" \
            -t directory

      - name: Find generated ZIP files
        id: find-zips
        run: |
          # Find all ZIP files in the module output directory
          ZIPS=$(find veraout/${{ matrix.module }}/ -type f \( -name "*.zip" -o -name "*.tar.gz" \) -printf '%f\n' | sort)
          
          if [ -z "$ZIPS" ]; then
            echo "::error::No ZIP files found for module ${{ matrix.module }}"
            exit 1
          fi
          
          # Convert to JSON array
          ZIP_ARRAY=$(printf '%s\n' "$ZIPS" | jq -R . | jq -c -s .)
          
          echo "Found ZIPs: $ZIP_ARRAY"
          echo "zip_files=$ZIP_ARRAY" >> $GITHUB_OUTPUT

      - name: Upload packaged module artifacts
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: veracode-${{ matrix.module }}
          path: veraout/${{ matrix.module }}/
          retention-days: 1

      - name: Output ZIP files per module
        id: output-zips
        run: |
          ZIP_LIST='${{ steps.find-zips.outputs.zip_files }}'
          echo "zip_files_${{ matrix.module }}=$ZIP_LIST" >> $GITHUB_OUTPUT
```

- [ ] **Step 3: Add outputs to the package-modules job header**

Modify the job definition to expose the ZIP file outputs. Find the `package-modules:` job definition and replace:

```yaml
  package-modules:
    name: Package Modules
    needs: detect-changes
    if: needs.detect-changes.outputs.modules != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: ${{ fromJson(needs.detect-changes.outputs.modules) }}
    steps:
```

With:

```yaml
  package-modules:
    name: Package Modules
    needs: detect-changes
    if: needs.detect-changes.outputs.modules != '[]'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: ${{ fromJson(needs.detect-changes.outputs.modules) }}
    outputs:
      zip_files_CMSCore: ${{ steps.output-zips.outputs.zip_files_CMSCore }}
      zip_files_CMSAdmin: ${{ steps.output-zips.outputs.zip_files_CMSAdmin }}
      zip_files_CMSSecurity: ${{ steps.output-zips.outputs.zip_files_CMSSecurity }}
      zip_files_CMSComponents: ${{ steps.output-zips.outputs.zip_files_CMSComponents }}
      zip_files_CMSInterfaces: ${{ steps.output-zips.outputs.zip_files_CMSInterfaces }}
      zip_files_CMSData: ${{ steps.output-zips.outputs.zip_files_CMSData }}
      zip_files_WebComponents: ${{ steps.output-zips.outputs.zip_files_WebComponents }}
      zip_files_Northwind: ${{ steps.output-zips.outputs.zip_files_Northwind }}
    steps:
```

- [ ] **Step 4: Commit the package-modules changes**

```bash
git add .github/workflows/veracode-multi-module-scan.yml
git commit -m "refactor: add ZIP file list output to package-modules job"
```

---

### Task 2: Refactor veracode-scan job - add combine phase

**Files:**
- Modify: `.github/workflows/veracode-multi-module-scan.yml` (veracode-scan job, add combine phase)

- [ ] **Step 1: Replace veracode-scan job header**

Find the `veracode-scan:` job definition (around line 117-123) and replace:

```yaml
  veracode-scan:
    name: Veracode Pipeline Scan
    needs: [ package-modules, detect-changes ]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        module: ${{ fromJson(needs.detect-changes.outputs.modules) }}
    steps:
```

With:

```yaml
  combine-zips:
    name: Combine ZIP Files
    needs: [ package-modules, detect-changes ]
    runs-on: ubuntu-latest
    outputs:
      all_zips: ${{ steps.combine.outputs.all_zips }}
    steps:
      - name: Combine ZIP files from all modules
        id: combine
        run: |
          # Array to hold all ZIP paths
          declare -a ALL_ZIPS
          
          # Process each module and its ZIP files
          MODULES='${{ needs.detect-changes.outputs.modules }}'
          
          for MODULE in $(echo "$MODULES" | jq -r '.[]'); do
            # Get the ZIP files output for this module
            ZIP_FILES_KEY="zip_files_${MODULE}"
            ZIP_FILES=$(eval "echo \${{ needs.package-modules.outputs.$ZIP_FILES_KEY }}")
            
            if [ -z "$ZIP_FILES" ] || [ "$ZIP_FILES" = "null" ]; then
              echo "::warning::No ZIPs found for module $MODULE"
              continue
            fi
            
            # Add each ZIP with its full path
            for ZIP in $(echo "$ZIP_FILES" | jq -r '.[]'); do
              ALL_ZIPS+=("veraout/$MODULE/$ZIP")
            done
          done
          
          # Convert to JSON array
          if [ ${#ALL_ZIPS[@]} -eq 0 ]; then
            ALL_ZIPS_JSON="[]"
          else
            ALL_ZIPS_JSON=$(printf '%s\n' "${ALL_ZIPS[@]}" | jq -R . | jq -c -s .)
          fi
          
          echo "Combined ZIPs: $ALL_ZIPS_JSON"
          echo "all_zips=$ALL_ZIPS_JSON" >> $GITHUB_OUTPUT

  veracode-scan:
    name: Veracode Pipeline Scan
    needs: [ package-modules, detect-changes, combine-zips ]
    runs-on: ubuntu-latest
    strategy:
      matrix:
        zip_file: ${{ fromJson(needs.combine-zips.outputs.all_zips) }}
    steps:
```

- [ ] **Step 2: Replace veracode-scan steps**

Replace all the existing steps in the veracode-scan job (lines 125-207) with:

```yaml
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Extract module from ZIP path
        id: extract-module
        run: |
          # Extract module name from zip_file (e.g., "veraout/CMSCore/app.zip" -> "CMSCore")
          MODULE=$(echo '${{ matrix.zip_file }}' | cut -d'/' -f2)
          echo "module=$MODULE" >> $GITHUB_OUTPUT
          echo "Extracted module: $MODULE from ${{ matrix.zip_file }}"

      - name: Download packaged artifacts
        uses: actions/download-artifact@v4
        with:
          name: veracode-${{ steps.extract-module.outputs.module }}
          path: veraout/${{ steps.extract-module.outputs.module }}/

      - name: Verify ZIP file exists
        run: |
          if [ ! -f "${{ matrix.zip_file }}" ]; then
            echo "::error::ZIP file not found: ${{ matrix.zip_file }}"
            ls -la veraout/${{ steps.extract-module.outputs.module }}/ || echo "Directory does not exist"
            exit 1
          fi
          echo "Verified ZIP file: ${{ matrix.zip_file }}"
          ls -lh "${{ matrix.zip_file }}"

      - name: Run Veracode Pipeline Scan
        uses: veracode/Veracode-pipeline-scan-action@esd-true
        with:
          vid: ${{ secrets.VERACODE_API_ID }}
          vkey: ${{ secrets.VERACODE_API_KEY }}
          file: ${{ matrix.zip_file }}
          fail_on_severity: critical
          fail_build: true
```

- [ ] **Step 3: Verify the job dependencies**

The updated workflow should have:
- `combine-zips` depends on: `package-modules` and `detect-changes`
- `veracode-scan` depends on: `package-modules`, `detect-changes`, and `combine-zips`

Run: `grep -E "needs:|name:" .github/workflows/veracode-multi-module-scan.yml | tail -20`

Expected: Output shows the three jobs with correct dependency order.

- [ ] **Step 4: Commit the veracode-scan refactoring**

```bash
git add .github/workflows/veracode-multi-module-scan.yml
git commit -m "refactor: implement per-ZIP matrix scanning with combine phase"
```

---

### Task 3: Validate workflow syntax and structure

**Files:**
- Verify: `.github/workflows/veracode-multi-module-scan.yml`

- [ ] **Step 1: Validate YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/veracode-multi-module-scan.yml'))" && echo "YAML is valid"
```

Expected: Output "YAML is valid" with no errors.

- [ ] **Step 2: Check job names and dependencies**

```bash
grep -E "^  [a-z-]+:" .github/workflows/veracode-multi-module-scan.yml
```

Expected: Output shows four jobs: `detect-changes`, `package-modules`, `combine-zips`, `veracode-scan`

- [ ] **Step 3: Verify matrix configurations**

```bash
grep -B2 "matrix:" .github/workflows/veracode-multi-module-scan.yml
```

Expected: Output shows:
- `package-modules` matrix: `module: ${{ fromJson(needs.detect-changes.outputs.modules) }}`
- `veracode-scan` matrix: `zip_file: ${{ fromJson(needs.combine-zips.outputs.all_zips) }}`

- [ ] **Step 4: Verify outputs are defined**

```bash
grep -A8 "package-modules:" .github/workflows/veracode-multi-module-scan.yml | grep -A8 "outputs:"
```

Expected: Output shows all `zip_files_*` outputs for each module.

- [ ] **Step 5: Check secrets are referenced correctly**

```bash
grep -c "secrets.VERACODE_API_ID" .github/workflows/veracode-multi-module-scan.yml
grep -c "secrets.VERACODE_API_KEY" .github/workflows/veracode-multi-module-scan.yml
```

Expected: Each appears at least once in the veracode-scan job.

- [ ] **Step 6: Final validation commit**

```bash
git add .github/workflows/veracode-multi-module-scan.yml
git commit -m "test: validate per-ZIP matrix workflow syntax and structure"
```

---

## Self-Review Against Spec

**Spec coverage:**
- ✅ Job 1 (detect-changes): Unchanged, continues to output module list
- ✅ Job 2 (package-modules): Modified to output `zip_files_<module>` per matrix item
- ✅ Job 3a (combine-zips): NEW job that combines all ZIP lists into single `all_zips` array
- ✅ Job 3b (veracode-scan): Refactored to matrix over individual ZIP files
- ✅ Phase 1 (combine): Merge all per-module ZIP lists into flat array
- ✅ Phase 2 (scan): Matrix over individual ZIPs, download artifact, scan with pipeline-scan-action
- ✅ Data flow: Full paths constructed during combine phase, used in matrix
- ✅ Pipeline-scan-action: Used with vid, vkey, file, fail_on_severity, fail_build
- ✅ Failure handling: Fail on critical issues, workflow fails if any matrix job fails
- ✅ No PR annotations: pipeline-scan-action has default behavior

**Placeholder scan:**
- ✅ All steps contain actual code and commands
- ✅ All YAML snippets are complete and valid
- ✅ No "TBD" or "TODO" sections
- ✅ Exact file paths and commands provided

**Type consistency:**
- ✅ `zip_files_*` outputs used consistently
- ✅ `all_zips` array used in matrix
- ✅ Module extraction logic consistent
- ✅ ZIP path format consistent (veraout/<module>/<zip>)

---

Plan saved to `docs/superpowers/plans/2026-06-03-github-actions-veracode-per-zip-matrix-implementation.md`.

Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?