# DiaMate — Development Agent Instructions

## 1. Project Instructions

You are an AI software development agent working on **DiaMate**.

`PROJECT.md` is the single source of truth for the project's approved requirements, architecture, development phases, and progress.

- Read `PROJECT.md` before beginning development work, focusing on the sections relevant to the requested task.
- Follow the approved requirements and architecture. Work only within the currently authorized phase and task scope.
- You may independently make reasonable technical implementation decisions that do not conflict with `PROJECT.md`.
- Do not introduce, remove, or modify product requirements without explicit user approval.
- Report conflicts between the task, implementation, and specification rather than silently changing requirements.

### Section 37 — The Living Development Checklist

**Section 37 of `PROJECT.md` is the central, continuously maintained task board.**

- All development tasks must originate from Section 37. Work only on a task explicitly selected and authorized by the user.
- Work on one selected task at a time unless the user explicitly authorizes otherwise.
- Never move automatically to the next task.
- Implementing code or passing tests is **not** approval of completion.
- A task may be marked `[x]` **only after the user explicitly tests or reviews it and approves completion**. Until then, leave it `[ ]`.
- If additional work is discovered, explain the proposed task and where it belongs. Add it to Section 37 only after explicit user approval, initially marked `[ ]`. Do not start it without authorization.
- Once the user approves a completed task, mark it `[x]`, review the relevant `PROJECT.md` sections, and make minimal documentation updates needed to reflect already approved decisions and implementation.
- Any new or changed product requirement or architecture decision requires separate explicit approval. Never silently rewrite the specification.
- Preserve all unrelated content and checklist entries.

### Parent Tasks & Subtasks

- A parent task may be marked `[x]` only after all of its subtasks have been implemented, verified, and explicitly approved by the user.
- Once work begins on a parent task, continue working through its subtasks, one at a time, before moving to another parent task, unless the user explicitly authorizes otherwise.
- Each subtask requires its own implementation, verification, and explicit user approval before being marked `[x]`.
- Never automatically proceed to the next subtask. Wait for the user's authorization.
- Adding, removing, splitting, or restructuring subtasks requires explicit user approval.
- Preserve the parent–subtask hierarchy and existing completion statuses when updating Section 37.

### Task Size & Granularity

- Tasks in Section 37 of `PROJECT.md` must be small, focused, and independently verifiable.
- Each task should represent one clear development objective with an identifiable completion criterion.
- Prefer tasks that can be implemented, tested, and approved individually.
- Avoid combining several unrelated features or significant changes into a single task.
- If an existing checklist task is too large or complex, propose splitting it into smaller subtasks before implementation.
- Splitting or restructuring checklist tasks requires explicit user approval.
- Do not expand a task's scope during implementation without approval.
- Each task must remain `[ ]` until its implementation is verified and explicitly approved by the user.
- Favor incremental development with minimal changes between approval cycles.

## 2. Development Workflow

For each selected, authorized task:

1. Inspect the existing implementation, relevant documentation, files, and dependencies.
2. Understand the expected behavior before modifying code.
3. Make the smallest correct change that satisfies the task, using existing architecture and conventions.
4. Run relevant tests and verify affected functionality.
5. Report the changes, test results, and clear steps for the user to verify them.
6. Wait for the user's explicit approval. If their testing reveals issues, keep working on the **same task** until resolved and approved.
7. After approval, update Section 37 and necessary documentation, then stop until the user selects another task.

### Minimal Change Principle

- Prefer the smallest possible change that fully satisfies the approved task.
- Modify only files and components directly relevant to it.
- Preserve existing code, behavior, and architecture whenever possible.
- Do not rewrite entire files where targeted edits suffice.
- Avoid unrelated refactoring, new dependencies, extra features, unnecessary abstractions, or style-only changes.
- Do not delete or replace working functionality unless required and authorized.
- If broader changes are genuinely necessary, explain why and ask for approval first.
- Never sacrifice correctness, security, or maintainability merely to minimize changed lines.
- Apply the same minimal-change rule when updating `PROJECT.md`.

## 3. Code & Testing Rules

- Write clean, readable, maintainable code that follows the existing project structure and conventions.
- Respect the responsibility boundaries and safeguards defined in `PROJECT.md`.
- Prefer existing libraries and services over introducing dependencies.
- Add or update tests for relevant behavior changes.
- Run focused tests and broader checks when justified by the impact of the change.
- Verify that affected existing functionality still works.
- Do not change unrelated implementation merely to make tests pass.
- Never claim a test passed if it was not actually run. Report unavailable or failed checks accurately.

### Safety & Security

- Never expose API keys, passwords, tokens, or personal medical information.
- Use environment variables or appropriate secure configuration for secrets.
- Do not commit `.env` files or sensitive user data.
- Never bypass required user confirmation, uncertainty handling, or insulin-calculation safety restrictions.
- Follow all applicable safety and privacy requirements in `PROJECT.md`.

## 4. Git & Approval Rules

- You may inspect Git status, diffs, and history as needed.
- Do not run `git commit`, `git push`, `git pull`, `git merge`, `git reset`, or other potentially destructive or history-changing Git operations without explicit authorization.
- Do not discard, overwrite, or revert existing user changes.
- Request approval before broad refactoring, deleting unrelated files, introducing dependencies, changing major architecture, or altering product requirements.
- User approval of a checklist task does **not** automatically authorize a commit or push.
- Keep working-tree changes limited to the authorized task whenever possible.

## 5. Reporting

After implementing an authorized task, provide a concise report containing:

- **Summary:** What was implemented or changed.
- **Modified Files:** Files added, modified, or deleted.
- **Verification:** Tests/checks actually run and their results.
- **User Testing:** Clear instructions to verify the behavior.
- **Outstanding Issues:** Limitations, unresolved problems, or checks not performed.

Clearly distinguish **implementation ready for review** from **user-approved completion**. Do not mark the task `[x]` until the user explicitly approves it. After approval, report checklist and documentation updates, and do not automatically proceed to another task.
