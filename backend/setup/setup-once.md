# Architecture Setup: Run Once

This project adopted the backend architecture rules, and setup has not run yet.

**Before your first file change in this project, complete the steps below.** Read-only questions do not need setup. When setup is done, tell the user in one line, then continue with their task.

Paths are relative to `.claude/rules/backend-architecture/`.

1. **Check the profile.** Read `stack/stack-profile.md`. If it contains no `{{`, it is complete, so skip to step 6.
2. **Detect the stack.** Read the package manifests, lockfiles, config files, and source tree. Find the HTTP framework, validation library, database access, test tool, source root, and commands.
3. **Map each role.** For every role in `core/layers.md`, find its existing analogue: directory, file suffix, and code shape. Keep the project's names where they exist.
4. **Handle a new project.** If there is no code yet, infer the stack from the manifest. If there is no manifest either, ask the user which stack to use. Propose a mapping, and mark each proposed entry `(proposed)`.
5. **Write the stack files.**
   - Replace every `{{...}}` in `stack/stack-profile.md`, and delete its "This profile is blank" line.
   - For each role with a code shape worth keeping, create `stack/<role>.md` with a short real example. List it in the role map's "Read first" column. For a role with no stack file, list the `core/` file that covers it, so the column never names a file that does not exist.
   - Scope these files with `paths` frontmatter. Double-quote every glob, start it with `**/`, and use no braces, because a glob that fails to parse makes the rule load in every session:

     ```yaml
     ---
     paths:
       - "**/*.repository.ts"
     ---
     ```

   - Keep each file under 200 lines.
6. **Fill `project-context.md`.** Replace each `{{...}}` with facts found in the repository. Write `unknown, ask when relevant` for anything you cannot find. Never guess.
7. **Record the target, not the drift.** The profile describes what the rules require. If existing code breaks them, never write the violation down as allowed. List the main violations for the user in a short note.
8. **Retire this file.** Overwrite this file with exactly this line: `Architecture setup is complete.` Use a file write, not a delete command.

## If a write is refused

Files under `.claude/` are protected, so the user sees a permission prompt for these writes. If a write is refused:

1. Do not retry it, and do not try another way of writing the file.
2. Follow the content you meant to write for the rest of the session, and continue with the user's task.
3. End your final message with a section titled "Architecture setup not saved". Include the complete content of every file you could not write, never a summary, and say that setup will run again next session until these files are saved.
