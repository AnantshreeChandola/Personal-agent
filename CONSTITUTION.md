# Project Constitution (Non-Negotiables)

- Protected branches: `main` (no direct pushes).
- All changes via Pull Requests with human review.
- **Green CI** (tests) required before merge.
- **Preview ≠ Execute**:
  - Preview uses stubs/mocks only; never contacts real providers.
  - Execute runs only after approval and plan signature verification.
- No secrets in repo or logs; follow least-privilege for all creds.
- Specs live in `/spec`, plans in `/plans`, plugins in `/plugins`, tests in `/tests`.
