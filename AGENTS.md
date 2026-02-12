# Commit Message Conventions

Use Conventional Commit-style messages:

`<type>(<scope>): <short description>`

## Scope Rules

- Use a path-based scope that matches the changed area.
- Examples:
  - Changes in `dotnet/nuget/**` -> `dotnet/nuget`

## Examples

- `feat(dotnet/nuget): add sourcelink strict build argument`
- `fix(dotnet/nuget): improve pack change detection gating`
- `docs(dotnet/nuget): document commit scope rules`
- `refactor(dotnet/nuget): simplify preload event wiring`
- `ci(repo): align pipeline behavior across projects`
- `chore(repo): update local development tooling defaults`
