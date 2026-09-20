# Contributing

Thanks for improving Ansible Development Languages.

## Before you start

- Read the README and understand the target platform behavior.
- Keep changes focused and idempotent.
- Do not commit credentials, private inventory data, generated installers, or
  local tool caches.
- Preserve existing user changes in shared task files.

## Development workflow

1. Create a branch for the change.
2. Update the relevant role task, defaults, vars, or documentation.
3. Run the syntax check:

   ```bash
   ansible-playbook site.yml --syntax-check
   ```

4. Run `git diff --check` and inspect the complete diff.
5. Describe supported platforms, verification steps, and any limitations in
   the pull request.

When adding a language, update `supported_languages`, add platform dispatch and
idempotent installation tasks, document the language in README.md, and test each
platform path that is available to you.

## Pull requests

Use a clear title and explain the behavior change. Include screenshots only when
they add useful context. Keep unrelated formatting or refactoring out of the
same pull request.

By contributing, you agree that your contribution may be distributed under the
project's MIT License.
