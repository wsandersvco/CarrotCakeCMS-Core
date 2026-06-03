# GitHub Actions Workflow: Per-ZIP Matrix Veracode Scanning

**Date:** 2026-06-03  
**Project:** CarrotCakeCMS-Core  
**Objective:** Refactor the multi-module Veracode scanning workflow to use a per-ZIP matrix strategy instead of per-module grouping, enabling true parallelism and granular failure reporting.

---

## Overview

The workflow currently packages changed modules and submits them for scanning. This design shifts the scan phase from a module-based matrix to a ZIP-based matrix, allowing each individual ZIP file to be scanned in parallel with the official `veracode/Veracode-pipeline-scan-action`.

**Three-job pipeline:**
1. `detect-changes` - Identify changed modules (unchanged from original)
2. `package-modules` - Package modules, output individual ZIP file lists per module
3. `veracode-scan` - Combine ZIP lists, matrix over individual ZIPs, scan each in parallel

---

## Job 1: detect-changes

**No changes from original design.** This job continues to:
- Detect which modules have changes in the PR via git diff
- Output a JSON array of module names: `["CMSCore", "CMSAdmin"]`

---

## Job 2: package-modules

**Purpose:** Package each changed module and generate a list of ZIPs produced.

**Depends on:** `detect-changes`  
**Matrix:** Over module names from `detect-changes.outputs.modules`  
**Parallel execution:** One job per module

**Process per matrix job:**
1. Check out the repository
2. Install Veracode CLI via official install script
3. Run `veracode package` on the module:
   ```
   veracode package -s <module_folder> -a -o veraout/<module_folder> -t directory
   ```
4. Find all generated ZIP files in `veraout/<module_folder>/`:
   ```bash
   find veraout/<module_folder>/ -type f \( -name "*.zip" -o -name "*.tar.gz" \) -printf '%P\n'
   ```
5. **Output job output `zip_files_<module>`** as JSON array of relative paths:
   - For CMSCore: `zip_files_CMSCore = ["app.zip", "symbols.zip"]` (relative to `veraout/CMSCore/`)
   - This allows combining outputs later while maintaining module association during packaging
6. Upload all ZIPs from `veraout/<module_folder>/` as artifact `veracode-<module_folder>` for retention

**Output format:**
- `zip_files_<module>`: JSON array of ZIP filenames (not full paths, just names within the module folder)
- Example: `zip_files_CMSCore = ["app.zip", "symbols.zip"]`

**Failure handling:** If packaging fails for any module, that matrix job fails, preventing scan job from running.

---

## Job 3: veracode-scan

**Purpose:** Scan all ZIPs from all modules in parallel using the official pipeline-scan-action.

**Depends on:** `package-modules`  
**Execution:** Two phases (combine then matrix)

### Phase 1: Combine Outputs

A non-matrix preparation step runs once:
1. Access all `zip_files_*` outputs from `package-modules` via the `needs` context
2. For each known module (from `detect-changes.outputs.modules`), retrieve its `zip_files_<module>` output
3. For each module's ZIP list, prepend the module folder path: transform `"app.zip"` → `"veraout/CMSCore/app.zip"`
4. Merge all transformed paths into a single flat JSON array
5. Output as `all_zips` for use in the matrix

**Bash logic:**
```bash
# For each module, get its zip_files output and prepend path
COMBINED=()
for MODULE in $MODULES; do
  ZIP_LIST=$(echo "${{ needs.package-modules.outputs[format('zip_files_{0}', MODULE)] }}")
  for ZIP in $(echo "$ZIP_LIST" | jq -r '.[]'); do
    COMBINED+=("veraout/$MODULE/$ZIP")
  done
done
# Convert to JSON array and output
ALL_ZIPS=$(printf '%s\n' "${COMBINED[@]}" | jq -R . | jq -c -s .)
echo "all_zips=$ALL_ZIPS" >> $GITHUB_OUTPUT
```

### Phase 2: Scan Matrix

Matrix over `all_zips`:
1. For each ZIP file path in the matrix:
   - Check out the repository
   - **Download the artifact containing that ZIP:**
     - Artifact name: `veracode-<module_folder>` (extracted from ZIP path)
     - Download to `veraout/<module_folder>/`
   - Run `veracode/Veracode-pipeline-scan-action@v1.0.11` with:
     - `vid`: `${{ secrets.VERACODE_API_ID }}`
     - `vkey`: `${{ secrets.VERACODE_API_KEY }}`
     - `file`: `${{ matrix.all_zips }}` (the ZIP path from matrix)
     - `fail_on_severity`: `"critical"` (or per policy)
     - `fail_build`: `true`

**Failure behavior:**
- If any scan detects critical issues, that matrix job fails
- If any matrix job fails, the entire `veracode-scan` job fails
- No PR annotations or comments are generated

---

## Data Flow Example

**Scenario:** PR changes CMSCore and CMSAdmin modules.

1. `detect-changes` outputs: `modules = ["CMSCore", "CMSAdmin"]`

2. `package-modules` matrix runs 2 jobs in parallel:
   - **CMSCore job:**
     - Packages CMSCore
     - Finds `veraout/CMSCore/app.zip` and `veraout/CMSCore/symbols.zip`
     - Outputs: `zip_files_CMSCore = ["app.zip", "symbols.zip"]`
     - Uploads artifact `veracode-CMSCore`
   
   - **CMSAdmin job:**
     - Packages CMSAdmin
     - Finds `veraout/CMSAdmin/admin.zip`
     - Outputs: `zip_files_CMSAdmin = ["admin.zip"]`
     - Uploads artifact `veracode-CMSAdmin`

3. `veracode-scan` combine step:
   - Reads: `zip_files_CMSCore = ["app.zip", "symbols.zip"]`, `zip_files_CMSAdmin = ["admin.zip"]`
   - Transforms to full paths: `["veraout/CMSCore/app.zip", "veraout/CMSCore/symbols.zip", "veraout/CMSAdmin/admin.zip"]`
   - Outputs: `all_zips = ["veraout/CMSCore/app.zip", "veraout/CMSCore/symbols.zip", "veraout/CMSAdmin/admin.zip"]`

4. `veracode-scan` matrix runs 3 jobs in parallel:
   - Job 1: Scans `veraout/CMSCore/app.zip`
   - Job 2: Scans `veraout/CMSCore/symbols.zip`
   - Job 3: Scans `veraout/CMSAdmin/admin.zip`

---

## Benefits & Trade-offs

**Benefits:**
- **True parallelism:** If one ZIP takes 10 minutes, others scan simultaneously
- **Granular reporting:** Know exactly which ZIP failed and why
- **Official action:** Uses `veracode/Veracode-pipeline-scan-action` (supported, documented)
- **Cleaner per-job responsibility:** One module packaged per job, one ZIP scanned per job
- **Scalable:** Works regardless of number of modules or ZIPs per module

**Trade-offs:**
- Output combining logic is more complex than original design
- More GitHub Actions API calls (per-ZIP matrix + per-ZIP artifact download)
- Verbose output merging in the combine step

---

## Implementation Notes

**GitHub Actions limitations:**
- Output keys in matrix jobs are not dynamically namespaced; using `zip_files_<module>` requires knowledge of module names in the combine step
- The combine step uses the `needs.<job>.outputs[format(...)]` syntax to access dynamic keys
- All artifacts must exist before the matrix runs, so combine step runs before the scan matrix is triggered

**Veracode pipeline-scan-action:**
- Accepts `file` parameter as a single ZIP path (not a list)
- Supports `fail_build: true` and `fail_on_severity: "critical"`
- Requires `vid`, `vkey`, and `file` as minimum inputs

---

## Success Criteria

- ✅ Changed modules are correctly identified via diff
- ✅ Each module is packaged and all generated ZIPs are captured
- ✅ `package-modules` outputs ZIP lists per module
- ✅ Combine step merges all ZIP lists into a flat array
- ✅ Scan matrix runs over individual ZIPs
- ✅ Each ZIP is scanned in parallel using the official action
- ✅ Workflow fails if any scan detects critical issues
- ✅ No PR annotations or check outputs are generated
- ✅ Granular failure reporting: know which ZIP failed

