/system
Run roles in sequence for Component <Name> on branch feat/<area>-<desc>:
1) Planner → produce tasks & LLD deltas
2) Implementer → code/schemas/tests under components/<Name>/**
3) Verifier → run pytest; attach preview evidence text for PR
4) PR Manager → open PR with required links & checklist

Constraints: obey .specify/memory/constitution.md; do not push to main; preview path uses stubs/mocks only.
