---
description: Create features from implementation plan.
---

Convert the implementation plan above into a structured feature breakdown. Follow these steps precisely:

## 1. Directory Setup

- Create `docs/features/` if it doesn't exist
- Create a sub-folder using kebab-case based on the plan name (e.g., `docs/features/user-authentication/`)

## 2. Copy Source Plan

- Save the original implementation plan as `_source-plan.md` in the sub-folder

## 3. Create Overview File

Create `00-overview.md` with this structure:

```md
# [Feature Set Name]

## Summary

[Brief description of what this implementation achieves]

## Features

| #   | Feature | File                 | Status         | Dependencies |
| --- | ------- | -------------------- | -------------- | ------------ |
| 1   | [Name]  | `01-feature-name.md` | ⬜ Not Started | None         |
| 2   | [Name]  | `02-feature-name.md` | ⬜ Not Started | #1           |

## Status Legend

- ⬜ Not Started
- 🟡 In Progress
- ✅ Complete
- ⛔ Blocked

## Implementation Order

[Recommended sequence based on dependencies]
```

## 4. Create Individual Feature Files

For each feature, create a numbered file (e.g., `01-feature-name.md`) with:

```md
# Feature: [Name]

## Overview

[What this feature does and why]

## Dependencies

- **Requires:** [List prerequisite features by number, or "None"]
- **Blocks:** [List features that depend on this one]

## Acceptance Criteria

- [ ] [Criterion 1]
- [ ] [Criterion 2]

## Technical Details

[Implementation specifics from the plan]

## Files to Create/Modify

- `path/to/file.ts` - [Purpose]

## Notes

[Any additional context or considerations]
```

## Guidelines

- Extract **atomic, independently testable** features
- Identify implicit dependencies between features
- Preserve all technical details from the source plan
- Use clear, action-oriented feature names
