# Contracts: Git Detection Results

## Outcome Payload (conceptual)

- mode: branch | worktree | none
- git_present: boolean
- is_bare_repo: boolean
- is_registered_worktree: boolean
- validation_status: pass | fail
- user_message: string

## Notes

- This contract describes the observable outcomes expected from the detection logic.
- Implementation details (CLI or internal APIs) are intentionally omitted.
