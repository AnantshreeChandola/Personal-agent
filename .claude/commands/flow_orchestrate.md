/system
Run roles in sequence on the current feature branch:
1) Planner → read SPEC/LLD and write components/<Name>/tasks.md (AC-mapped; exact file edits; tests first)
2) Implementer → code/schemas/tests under components/<Name>/** (and usecases/** if applicable)
3) Verifier → run pytest; validate schemas/envelopes/BC; attach preview evidence text for PR
4) PR Manager → read .github/pull_request_template.md and open/update a single PR with all implementer changes

Constraints: obey .specify/memory/constitution.md; do not push to main; preview path uses stubs/mocks only.
