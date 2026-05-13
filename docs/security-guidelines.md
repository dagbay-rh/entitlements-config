# Security Guidelines for entitlements-config

## Overview

This repository defines SKU-to-entitlement mappings loaded as Kubernetes ConfigMaps into the entitlements service. Changes here directly control which Red Hat products and features customers can access. A misconfigured bundle can silently grant or revoke access at scale.

## Repository Structure

Four environment directories under `configs/`, each containing a `bundles.yml`:

- `configs/prod/` -- Production (commercial cloud)
- `configs/stage/` -- Staging (commercial cloud)
- `configs/fedramp-prod/` -- FedRAMP production
- `configs/fedramp-stage/` -- FedRAMP staging

Each file is a Kubernetes `Template` wrapping a `ConfigMap` named `entitlements-config`.

## Bundle Definition Types

Every bundle must use exactly one of these entitlement mechanisms:

| Mechanism | Effect | Risk Level |
|---|---|---|
| `skus` | Entitled if account holds any listed SKU; no trial tracking | Low |
| `eval_skus` + `paid_skus` | Entitled with `is_trial` flag support | Medium -- SKU miscategorization affects billing signals |
| `use_valid_org_id: true` | Entitled for any account with a valid org ID | High -- grants broad access |
| `use_valid_org_id: false` | Entitled for everyone unconditionally | Critical -- universal access |
| `use_is_internal: true` | Entitled only for Red Hat internal accounts | High -- gates internal-only tooling |

## Critical Rules

### 1. Never mix entitlement mechanisms in a single bundle

A bundle must use one mechanism. Do not combine `skus` with `use_valid_org_id`, or `eval_skus` with `skus`.

### 2. Never place eval/trial SKUs in `paid_skus` (or vice versa)

Trial SKUs (typically prefixed `RH00`, `SER057`, `SER076`, `SER079`) must go in `eval_skus`. Placing them in `paid_skus` causes `is_trial` to report `false` for trial accounts, which is a billing and compliance error. See commit `a0c2322` for a real instance of this mistake being corrected.

### 3. The `internal` bundle must only use `use_is_internal: true`

This bundle gates access to internal-only Red Hat features. Never add SKUs to it, and never change its mechanism. It must exist in all four environment files.

### 4. Avoid setting `use_valid_org_id: false` without explicit approval

This grants the entitlement to every request regardless of account status. Currently `openshift` in prod, fedramp-prod, and fedramp-stage uses this mechanism (the stage `openshift` bundle uses a `skus:` list instead). Adding new bundles with this flag or changing existing bundles to it is a privilege escalation risk.

### 5. Stage must be updated before prod

Changes should land in `configs/stage/bundles.yml` first, then `configs/prod/bundles.yml` after validation. Never push to prod without a corresponding stage change already merged.

### 6. FedRAMP environments have a reduced bundle set

FedRAMP configs intentionally contain fewer bundles (for example, `rhods`, `rhoam`, `rhosak`, and `acs` are present in commercial environments but not in FedRAMP) and do not use `eval_skus`/`paid_skus` separation (they use flat `skus` lists). Do not add bundles to FedRAMP that do not exist in commercial environments. Do not add the eval/paid split to FedRAMP unless explicitly directed.

### 7. Do not remove bundles without verifying downstream dependencies

Commit history shows multiple revert cycles (`#95`, `#96`, `#97`, `#99`) where bundle removals broke stage pods. Before removing any bundle, confirm that no active service depends on it. Removing a bundle name entirely causes the entitlements API to stop returning it, which can break consumers that expect the key to exist.

### 8. Keep bundles consistent across environment pairs

`prod` and `stage` should have the same bundle names. `fedramp-prod` and `fedramp-stage` should be identical or near-identical. A bundle present in one but missing from its pair indicates an incomplete rollout.

## SKU Conventions

- SKU identifiers are alphanumeric codes like `MCT3691`, `MW02577`, `SER0496`
- Common suffixes: `F3` (3-year term), `F5` (5-year), `RN` (renewal), `S` (support), `MO` (monthly), `HR` (hourly)
- A base SKU and all its variants (e.g., `MCT3691`, `MCT3691F3`, `MCT3691RN`, `MCT3691S`) should appear in the same bundle and same category (eval or paid)
- `RH00xxx` SKUs are typically evaluation/trial SKUs and appear in `eval_skus` in commercial environments; in FedRAMP they appear in flat `skus` lists where the eval/paid distinction does not apply
- `SER0xxx` SKUs span both trial and paid uses depending on range and context: for example, `SER0569`, `SER0570`, `SER0764`, `SER0795`, and `SER0798` are used as eval SKUs in commercial environments, while others such as `SER0496` and `SER0497` appear in `paid_skus` lists. Confirm the category with the product team before placement
- `SVC` prefixed SKUs are paid service SKUs

## Adding SKUs Checklist

1. Confirm the SKU category (eval vs. paid) with the product team
2. Add to `eval_skus` or `paid_skus` accordingly (not the wrong list)
3. Include all term/renewal/support variants of the SKU
4. Add to stage first, then prod after validation
5. For FedRAMP bundles, add to the flat `skus` list (no eval/paid split)
6. Do not add a SKU to multiple bundles unless explicitly required -- a SKU appearing in two bundles grants both entitlements

## YAML Formatting Rules

- The bundles list is embedded in a YAML multiline string (`bundles.yml: |`) inside the ConfigMap data field
- Indentation is critical: bundle entries are indented 6 spaces from the left margin (under the `|` block scalar)
- SKU entries are indented 10 spaces (under `skus:`, `eval_skus:`, or `paid_skus:`)
- Invalid YAML will cause the ConfigMap to fail to load, breaking all entitlements for the environment
- Preserve the `qontract.recycle: "true"` annotation -- it enables automatic ConfigMap rotation

## Pre-commit Hooks

The repository uses `rh-multi-pre-commit` (Red Hat's leak detection tool). This scans for accidentally committed secrets and credentials. It does not validate bundle structure or SKU correctness.

## Deployment

Merging a PR does not deploy changes automatically. The SHA reference in app-interface must be updated to the new commit for each target environment. This is a manual step that provides a deployment gate.

## Review Checklist for PRs

- [ ] No eval/trial SKUs in `paid_skus` lists
- [ ] No paid SKUs in `eval_skus` lists
- [ ] No new `use_valid_org_id: false` bundles without justification
- [ ] `internal` bundle unchanged (still `use_is_internal: true` only)
- [ ] Stage updated alongside or before prod
- [ ] FedRAMP files updated if the bundle exists there
- [ ] All SKU variants included (base + F3/F5/RN/S/MO/HR)
- [ ] No duplicate SKU entries within a single list
- [ ] Valid YAML (proper indentation under the block scalar)
- [ ] Bundle names match across environment pairs
