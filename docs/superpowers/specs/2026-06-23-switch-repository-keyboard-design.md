# Switch Repository Keyboard Access Design

## Goal

Allow users to switch the focused Git Graph repository without using the mouse.

## Requirements

- The existing Git Graph repository dropdown remains the primary in-webview repository switcher.
- After opening the repository dropdown and filtering repositories by typing, `ArrowUp` and `ArrowDown` move through matching visible repositories.
- Pressing `Enter` in the repository dropdown activates the currently highlighted matching repository.
- A command palette command named `Git Graph: Switch Repository...` opens a native VS Code QuickPick containing the repositories currently known to Git Graph.
- Selecting a repository from the command palette QuickPick opens or focuses Git Graph and loads that repository.
- Repository names and ordering must match Git Graph's existing repository dropdown behavior.
- The implementation must be small and upstream-friendly, without replacing the existing dropdown UI.
- Users must be able to build a local VSIX and install it if the upstream pull request is not released.

## Recommended Approach

Reuse existing repository-switching primitives:

- Extend the generic webview `Dropdown` component with keyboard navigation for filtered options.
- Add a focused extension command, `git-graph.switchRepository`, contributed as `Git Graph: Switch Repository...`.
- Reuse `RepoManager.getRepos()`, `getSortedRepositoryPaths()`, `getRepoName()`, and `GitGraphView.createOrShow()` in the command handler.

This keeps the feature close to existing code paths and avoids a repo-specific dropdown fork.

## Component Design

### Webview Dropdown

`web/dropdown.ts` owns dropdown rendering, filtering, and selection.

Add internal highlighted-option state for visible filtered options. When the filter text changes, initialize the highlight to the first visible option. While the filter input is focused:

- `ArrowDown` moves the highlight to the next visible option, wrapping at the end.
- `ArrowUp` moves the highlight to the previous visible option, wrapping at the beginning.
- `Enter` activates the highlighted option by reusing the existing option selection path.

The behavior should apply to the existing generic dropdown component, but must preserve current mouse and multi-select branch dropdown behavior.

### Command Palette

`src/commands.ts` owns command registration.

Add `git-graph.switchRepository` and implement it as a native QuickPick. The QuickPick items should use:

- `label`: configured repository name or short repository name.
- `description`: full repository path.
- sorted order: `getSortedRepositoryPaths(repos, getConfig().repoDropdownOrder)`.

When a repository is selected, call `GitGraphView.createOrShow()` with `{ repo: selectedPath }`.

For zero known repositories, open Git Graph normally so the existing no-repo flow is preserved. For one known repository, open that repository directly without forcing an unnecessary picker.

### Metadata And Docs

Update `package.json` command contributions and README command list. If a changelog entry is expected by upstream conventions, add a concise unreleased or top-entry bullet only if the repository already has that pattern.

## Testing

Add focused tests for the command palette path in `tests/commands.test.ts`:

- command is registered and counted in `CommandManager` construction expectations.
- multiple repos produce a QuickPick in existing repository dropdown order.
- selecting a repo calls `GitGraphView.createOrShow()` with the selected repo.
- no selection does not open a specific repo.
- one repo opens directly.
- no repos opens Git Graph with `null`.

For dropdown keyboard behavior, prefer a focused DOM-capable test if the current test setup supports it. If it does not, rely on TypeScript compile plus manual verification in an Extension Development Host.

## Verification

Run the applicable project checks:

- `npm install`
- `npm run lint`
- `npm test`
- `npm run compile`

Package and local install path:

- `npm install -g vsce` if `vsce` is not available.
- Use upstream's documented local flow: `npm run package-and-install`.
- Restart VS Code or Cursor and verify the installed Git Graph version is the local build.

## Out Of Scope

- Replacing Git Graph's repository dropdown with VS Code QuickPick.
- Adding custom keybindings beyond command palette access.
- Changing repository discovery or repository sorting semantics.
- Changing branch dropdown filtering semantics except where generic keyboard navigation necessarily shares dropdown code.
