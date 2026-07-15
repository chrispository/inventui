# Potential Improvements

The biggest opportunity is making the package list more trustworthy and actionable, rather than adding more package managers immediately.

## 1. Make loading and failures visible

Loading currently blocks before the first frame is drawn, and `loading_progress` never receives progress (`src/main.rs:41`, `src/app.rs:187`). Collectors also return an empty list for both “no packages” and “command failed.”

Potential improvements:

- Run collection asynchronously so the interface can render while packages are loading.
- Show each source as loading, loaded, unavailable, or failed.
- Include useful failure messages, such as a missing executable or unsuccessful command.
- Distinguish an empty package source from a source that could not be queried.
- Make reload progress visible as each collector finishes.

For example, missing `expac` should not silently make every pacman package disappear.

## 2. Clarify what “Explicit” means across ecosystems

The current filter only includes packages whose install reason is exactly `"explicit"` (`src/app.rs:273`), so it effectively excludes every cargo, npm, and pip row.

However:

- Globally installed npm packages are generally user-requested.
- Entries from `cargo install` are inherently user-requested.
- Pip packages may be top-level requests, dependencies, or impossible to classify reliably.

Potential improvements:

- Replace `Option<String>` with an install-reason enum such as `Explicit`, `Dependency`, and `Unknown`.
- Treat global npm and cargo packages as explicit.
- For pip, consider using top-level or “not required” packages as an approximation while clearly marking the result as inferred.
- Consider renaming the filter to “User installed” if that better represents the intended behavior.
- Surface unknown classification rather than silently excluding those packages.

## 3. Track package scope and use it for safe actions

A source such as “pip” is not sufficient to uniquely identify where a package is installed. It may belong to a particular Python interpreter, virtual environment, user installation, or system installation. Similar ambiguity exists for npm prefixes and `CARGO_HOME`.

Potential improvements:

- Store the environment or installation scope on every package.
- Display that scope in the table or details panel.
- Record the exact executable or environment that discovered each package.
- Use that same executable and environment when uninstalling it.
- Include scope as part of the package’s stable identity.

This prevents a changed shell environment from removing a different package than the one originally displayed.

## 4. Strengthen uninstall safety and feedback

The current confirmation dialog is visually good, but the subprocess result is discarded and the package list is always reloaded (`src/main.rs:177`). Pacman’s `-Rns` can also remove more than the selected package.

Potential improvements:

- Show the exact command that will run.
- Preview packages that may also be removed, where the package manager supports it.
- Require stronger confirmation for Omarchy-managed or otherwise protected packages.
- Report whether the operation succeeded, failed, or was cancelled.
- Preserve useful stderr or exit-status information.
- Only report successful removal when the command actually succeeds.
- Consider allowing the user to choose between conservative and dependency-cleaning removal behavior.

## 5. Make the details panel navigable

Long dependency lists can extend below the popup, but currently any key closes the details panel (`src/main.rs:116`).

Potential improvements:

- Scroll with `j`/`k` and the arrow keys.
- Support `PageUp`, `PageDown`, `Home`, and `End`.
- Close only with an explicit key such as `Esc` or `q`.
- Allow opening the package homepage from the details panel.
- Show a scroll indicator when more content exists below the visible area.
- Cache recently fetched details so reopening the panel is immediate.

## 6. Preserve the selected package through sorting and filtering

Selection is currently represented by a visible row number. After sorting or filtering, that same row can refer to a different package.

Potential improvements:

- Define a stable package identity using source, scope, and package name.
- Remember the selected identity before sorting, filtering, or reloading.
- Relocate the selection to that package afterward when it is still visible.
- Choose a predictable neighboring row when the selected package is filtered out or removed.
- Retain the current search query when `/` is pressed so it can be edited rather than always replaced.
- Consider a direct source/filter picker once cycling through options becomes cumbersome.

## 7. Improve narrow-terminal and Unicode behavior

The table currently uses fixed column widths. The truncation helper also slices Rust strings by byte offset (`src/ui.rs:589`), which can panic when a non-ASCII package name is cut inside a multi-byte character.

Potential improvements:

- Make column widths responsive to the available terminal width.
- Hide or collapse lower-priority columns on narrow terminals.
- Use Unicode-aware display width and truncation.
- Avoid showing `0.00 GB` for small packages; use adaptive units such as KiB, MiB, and GiB.
- Reconsider the `Site` column, whose information often overlaps with Source.
- Potentially replace `Site` with more actionable information such as Scope or Install Reason.
- Adapt the help row when the terminal is too narrow to show every shortcut.

## 8. Add inventory history and comparison

A snapshot and diff feature could become a strong differentiator for the program.

Potential improvements:

- Save a snapshot of the current inventory.
- Compare the current state with a previous snapshot.
- Report packages that were added, removed, or upgraded.
- Export inventory data as JSON or CSV.
- Detect the same package name across multiple ecosystems or scopes.
- Add views for outdated packages and orphan candidates.
- Allow comparisons suitable for rebuilding or auditing another machine.

An example summary might be: “12 packages added, 3 removed, and 8 upgraded since the previous snapshot.”

## 9. Add tests around pure logic and collector parsing

There currently appear to be no automated tests.

High-value test coverage would include:

- Collector parsing using saved command-output fixtures.
- Empty, malformed, and partial subprocess output.
- Filter combinations and explicit-package semantics.
- Sorting when size or install date is missing.
- Stable selection through filtering, sorting, and reloads.
- Package-manager command generation.
- Unicode-aware truncation.
- Details parsing for pacman and pip continuation lines.
- Scoped npm packages.
- Omarchy manifest parsing.

A small command-runner abstraction would make collectors testable without invoking package managers installed on the test machine.

## 10. Make package-source capabilities more data-driven

The current separation between collectors, application state, details, and rendering is solid. However, knowledge about each source is spread across several enums and match statements.

If more package sources are expected, a source descriptor could group together:

- Identifier and display label.
- Color and table presentation.
- Availability detection.
- Collection behavior.
- Supported metadata fields.
- Details lookup.
- Uninstall behavior.
- Whether install reason, size, dates, or reverse dependencies are available.

This would reduce the number of separate files and match statements that need to change when adding another package manager.

## Suggested implementation sequence

1. Trustworthy collection, progress, and per-source error reporting.
2. Consistent install-reason and package-scope semantics.
3. Safer uninstall commands and accurate result reporting.
4. Scrollable details and stable selection behavior.
5. Responsive and Unicode-safe rendering.
6. Snapshot, diff, and export functionality.
7. A more data-driven source architecture when additional collectors justify it.

Following this order would evolve the program from a polished package table into a package inventory that users can trust for auditing and system cleanup.
