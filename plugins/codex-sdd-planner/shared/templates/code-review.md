# Code Review: {{TITLE}}

## Metadata
- **Review file:** `{{REVIEW_FILE}}`
- **Created:** {{CREATED_DATE}}
- **Target repo:** `{{TARGET_REPO}}`
- **Plan:** `{{PLAN_PATH}}`
- **Phase:** `{{PHASE_PATH}}`
- **Diff scope:** {{DIFF_SCOPE}}
- **Review type:** Initial | Fix verification | Follow-up

## Overall Verdict
**Alignment:** Strong | Moderate | Weak
**Lane status:** OK | DEGRADED ({{DEGRADED_REASON}}) | BLOCKED ({{BLOCKED_REASON}})
**Critical issues:** {{CRITICAL_COUNT}}
**Top items to address:**
- {{TOP_ITEM_1}}
- {{TOP_ITEM_2}}
- {{TOP_ITEM_3}}

## Diff Scope
- **Commits reviewed:** {{COMMITS_REVIEWED}}
- **Files changed:** {{FILES_CHANGED_COUNT}}
- **Reviewers dispatched:** drift-detector, quality-scanner, spec-compliance, blind-spot-finder{{PROJECT_LANES_SUFFIX}}

## Project Review Lanes
<!-- Omit this section entirely if no project lanes were discovered. -->
- **Ran:** {{PROJECT_LANES_RAN}}
- **Skipped:** {{PROJECT_LANES_SKIPPED}}
- **Failed to dispatch:** {{PROJECT_LANES_FAILED}}
- **Errored:** {{PROJECT_LANES_ERRORED}}
- **Oversized:** {{PROJECT_LANES_OVERSIZED}}
- **Malformed:** {{PROJECT_LANES_MALFORMED}}

## Confirmed Findings
<!-- Findings agreed by two or more independent lanes. Use "None." if empty. -->

### {{SEVERITY}} - {{FINDING_TITLE}}
**Caught by:** {{LANE_NAMES}}
**Location:** `{{PATH}}:{{LINE}}`
**Detail:** {{DETAIL}}
**Recommendation:** {{RECOMMENDATION}}

## Disagreements
<!-- Reviewer contradictions are findings. Use "None." if empty. -->

### {{DISAGREEMENT_TITLE}}
- **{{LANE_A}} says:** {{LANE_A_POSITION}}
- **{{LANE_B}} says:** {{LANE_B_POSITION}}
- **What this means:** {{INTERPRETATION}}
- **Recommendation:** {{RECOMMENDATION}}

## Blind Spots Only `blind-spot-finder` Caught
<!-- Unique adversarial findings. Use "None." if empty. -->

### {{SEVERITY}} - {{FINDING_TITLE}}
**Location:** `{{PATH}}:{{LINE}}`
**Scenario:** {{SCENARIO}}
**Recommendation:** {{RECOMMENDATION}}

## Drift
<!-- Unique findings from drift-detector that were not confirmed or disagreed elsewhere. Use "None." if empty. -->

### {{SEVERITY}} - {{FINDING_TITLE}}
**Location:** `{{PATH}}:{{LINE}}`
**Plan reference:** {{PLAN_REFERENCE}}
**Issue:** {{ISSUE}}
**Recommendation:** {{RECOMMENDATION}}

## Quality
<!-- Unique findings from quality-scanner. Use "None." if empty. -->

### {{SEVERITY}} - {{FINDING_TITLE}}
**Lens:** {{QUALITY_LENS}}
**Location:** `{{PATH}}:{{LINE}}`
**Issue:** {{ISSUE}}
**Validated by:** {{VALIDATION}}
**Recommendation:** {{RECOMMENDATION}}

## Spec Compliance
<!-- Unique findings from spec-compliance. Use "None." if empty. -->

### {{SEVERITY}} - {{FINDING_TITLE}}
**Requirement:** {{REQUIREMENT_REFERENCE}}
**Location:** `{{PATH}}:{{LINE}}`
**Issue:** {{ISSUE}}
**Recommendation:** {{RECOMMENDATION}}

## Project Lanes
<!-- Omit this section entirely if no project lanes returned unique findings. One sub-heading per lane or grouped lane. -->

### {{PROJECT_LANE_NAME}}
{{PROJECT_LANE_FINDINGS}}

## Open Questions
<!-- Unverified suspicions that could not be cross-validated. Use "None." if empty. -->
- **{{SOURCE_LANE}}:** {{QUESTION}}

## Prior Review Verification
<!-- Include only for fix-verification reviews. Omit for initial reviews. -->
- **Prior review:** `{{PRIOR_REVIEW_FILE}}`
- **Overall status:** Fixed | Partially fixed | Still open | Superseded | Won't fix

### Finding Resolutions
- **{{PRIOR_FINDING_TITLE}}** - Fixed | Partial | Still open | Superseded | Won't fix
  - Evidence: {{EVIDENCE}}
  - Notes: {{NOTES}}

## Raw Reports

<details>
<summary>drift-detector report</summary>

{{DRIFT_DETECTOR_REPORT}}
</details>

<details>
<summary>quality-scanner report</summary>

{{QUALITY_SCANNER_REPORT}}
</details>

<details>
<summary>spec-compliance report</summary>

{{SPEC_COMPLIANCE_REPORT}}
</details>

<details>
<summary>blind-spot-finder report</summary>

{{BLIND_SPOT_FINDER_REPORT}}
</details>

{{PROJECT_LANE_RAW_REPORTS}}

---

## Resolution
**Status:** Pending
**Resolved in:** Not yet resolved
**Verified by:** Not yet verified

### Finding Resolutions
- Pending. Add resolution entries here after fixes are implemented and verified by a later review.
