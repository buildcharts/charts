# Commit Message Conventions

Use Conventional Commit-style messages:

`<type>(<scope>): <short description>`

## Allowed Types

- `fix`
- `refactor`

## Scope Rules

- Use a path-based scope that matches the changed area.
- Examples:
  - Changes in `dotnet/nuget/**` -> `dotnet/nuget`
  - Changes in `dotnet/**` -> `dotnet`
  - Changes in `electron/**` -> `electron`
  - Cross-cutting changes in multiple top-level areas -> `repo`

## Examples

- `fix(dotnet/nuget): improve pack change detection gating`
- `refactor(electron): simplify preload event wiring`
- `fix(repo): align ci script behavior across projects`
