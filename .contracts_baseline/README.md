# Contract Baseline Mechanism

> **Purpose**: Detect breaking changes in API contracts automatically  
> **Version**: 1.0  
> **Created**: 2025-11-09 (Phase 14.0)  
> **Language**: English

---

## What is Contract Baseline?

Contract baselines are snapshots of API contracts (CONTRACT.md, contract.json) stored at specific points in time. By comparing current contracts with baselines, we can automatically detect breaking changes.

---

## How It Works

### 1. Create Baseline
```bash
make update_baselines

# Creates/updates baseline files:
# .contracts_baseline/modules_<module>_CONTRACT.md.baseline
# .contracts_baseline/tools_<tool>_contract.json.baseline
```

### 2. Detect Changes
```bash
make contract_compat_check

# Compares current contracts with baselines
# Reports: added fields, removed fields, changed types
# Blocks: breaking changes (field removal, type changes)
```

### 3. Update Baseline
```bash
# After confirming changes are intentional:
make update_baselines

# This accepts the new contract as the new baseline
```

---

## Breaking Change Detection

### What is Considered Breaking?

**Breaking (Blocked)**:
- Removed fields
- Changed field types
- Removed endpoints
- Changed required/optional status (required -> optional OK, reverse is breaking)

**Non-Breaking (Allowed)**:
- Added new optional fields
- Added new endpoints
- Added new response fields
- Relaxed constraints

---

## Exemption Mechanism

### Skip Baseline Check

For intentional breaking changes (e.g., major version):

**Option 1: Update Baseline First**
```bash
make update_baselines
# Then make your changes
```

**Option 2: Emergency Override**
```bash
export SKIP_CONTRACT_CHECK=true
make contract_compat_check  # Will show warnings but not block
```

**Option 3: Document in Contract**
```markdown
# In CONTRACT.md, add:
## Breaking Changes in v2.0
- Removed field `old_field` (deprecated in v1.5)
- Changed `user_id` from string to integer
- Migration guide: See MIGRATION_V2.md
```

---

## Workflow Integration

### During Development
```bash
# 1. Make contract changes
vim modules/user/doc/CONTRACT.md

# 2. Check compatibility
make contract_compat_check
# If breaking: Blocked with detailed report

# 3. If intentional breaking change:
#    a. Add migration guide
#    b. Update CHANGELOG.md with BREAKING CHANGE note
#    c. Update baseline: make update_baselines

# 4. Proceed with code changes
```

### In CI/CD
```yaml
# .github/workflows/ci.yml
- name: Contract Compatibility Check
  run: make contract_compat_check
  # Blocks PR if breaking changes detected without baseline update
```

---

## File Structure

```
.contracts_baseline/
├── README.md (this file)
├── modules_common_CONTRACT.md.baseline
├── modules_user_CONTRACT.md.baseline
├── tools_openapi_contract.json.baseline
└── ... (one baseline per contract file)
```

**Naming Convention**: `{type}_{module_or_tool}_{filename}.baseline`

---

## Best Practices

### 1. Update Baseline After Each Release
```bash
# After deploying v1.5:
git tag v1.5.0
make update_baselines
git add .contracts_baseline/
git commit -m "chore: update contract baselines for v1.5.0"
```

### 2. Document Breaking Changes
Always include in CHANGELOG.md:
```markdown
## [2.0.0] - 2025-11-09

### BREAKING CHANGES
- **user module**: Removed `old_field`, use `new_field` instead
- **auth module**: Changed `user_id` type from string to integer

### Migration Guide
See MIGRATION_V2.md for detailed migration steps.
```

### 3. Gradual Migration
For breaking changes, provide transition period:
```markdown
# v1.5: Deprecate old field
old_field: string (deprecated, use new_field)
new_field: string

# v2.0: Remove old field
new_field: string
```

---

## Commands

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `make update_baselines` | Create/update baselines | After release, before breaking changes |
| `make contract_compat_check` | Check compatibility | Before committing contract changes, in CI |
| `git diff .contracts_baseline/` | Review baseline changes | Before committing baseline updates |

---

## Troubleshooting

### Q: Baseline check failed but change is intentional?
**A**: Update baseline with `make update_baselines`, document in CHANGELOG.md

### Q: How to see what changed?
**A**: Run `make contract_compat_check` - it shows detailed diff

### Q: Can I disable baseline check temporarily?
**A**: Yes, `export SKIP_CONTRACT_CHECK=true`, but document reason in commit message

### Q: Baseline files in version control?
**A**: Yes, commit to git. They serve as contract history.

---

## See Also

- **Contract Compatibility Check**: scripts/contract_compat_check.py (implementation)
- **Contract Writing**: doc/modules/TEMPLATES/CONTRACT.md.template (template)
- **API Design**: doc/process/CONVENTIONS.md (guidelines)

---

**Version**: 1.0  
**Last Updated**: 2025-11-09  
**Maintained by**: Project Team
