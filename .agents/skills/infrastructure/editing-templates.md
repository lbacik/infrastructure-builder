# Editing a template

Part of the `infrastructure` skill ([SKILL.md](SKILL.md)). `<git-url>` and
`<ref>` are the current remote template source. Switch to local mode:

1. If `.gitignore` lists `templates/`, remove that entry - a submodule path
   must be trackable.
2. `git submodule add <git-url> templates/infrastructure`, then check out the
   ref the remote mode used.
3. Write `source: templates/infrastructure` into the `## Templates` section of
   `docs/resources.md`, creating the file or section if needed.
4. Make the template change inside `templates/infrastructure` and commit it
   there, in the submodule's own history. Commits of this repository carry
   only the updated submodule pointer, `.gitmodules`, `.gitignore`, and
   `docs/resources.md`.
