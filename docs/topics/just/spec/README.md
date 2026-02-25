# Just Specification

Spec for the Just command runner conventions. For the conventions themselves, see [README.md](../README.md).

## Purpose

The Just configuration has a non-obvious architecture: a four-layer dispatch system with provider abstraction, one-recipe-per-file granularity, and a coupling between recipe names and prompt directory names. Without documentation, an AI agent or engineer adding a recipe would not discover this structure from the `Justfile` alone and would likely add recipes in the wrong location or break the dispatch pattern.

## Decisions

- **Just as sole operational interface.** Repository operations must go through Just rather than invoking tools directly. Just decouples the developer and AI workflow from specific tool choices. If the package manager or linter changes, the Just recipes absorb the change while the commands remain the same. This also creates a single place to audit and control what tools are invoked. Alternative considered: allowing direct tool invocation alongside Just (rejected; defeats the abstraction and creates inconsistent workflows that diverge over time).

- **One recipe per file.** Each file has a single, unambiguous purpose. Diffs are granular. File names directly communicate what the file does. Alternative considered: grouping related recipes in one file (rejected; obscures individual recipe purpose and produces noisier diffs).

- **Four-layer architecture.** Root, AI dispatch, AI backends, and settings. This separates public interface from implementation, and implementation from provider-specific details. The dispatch layer exists so the developer interacts with a stable set of root commands regardless of which AI backend is active. Alternative considered: direct root-to-backend calls without dispatch (rejected; changing the active backend would require editing every root recipe instead of one variable).

- **Import for root, mod for everything else.** Root recipes are the primary interface. Using `import` surfaces them at the top level of `just -l` without a namespace prefix. AI modules use `mod` to namespace their recipes, preventing collisions between identically-named recipes across modules (e.g., a recipe like `save` can exist in root, ai, and provider modules without conflict). Alternative considered: `mod` for everything (rejected; forces users to type `just root build` instead of `just build`). Alternative considered: `import` for everything (rejected; creates recipe name collisions).

- **AI dispatch via variable.** The `ai_bin` variable in `settings.just` controls which backend receives AI commands. Changing the active AI provider is a single-line edit. Alternative considered: environment variable (rejected; dotenv is already loaded but the choice is a project-level default, not a per-session override). Alternative considered: command-line argument on each call (rejected; adds friction to every invocation).

- **Private AI recipes.** AI module recipes are implementation details. Marking them `[private]` keeps `just -l` output clean and communicates that developers interact through root commands. The namespaced modules (`just <provider> <recipe>`) remain accessible for debugging or direct use.

- **Managed shell for all recipes.** Configuring `set shell` in `settings.just` ensures every recipe runs in a managed environment with consistent tool versions. Alternative considered: relying on PATH (rejected; breaks when managed tools are not globally installed). Alternative considered: prefixing individual commands with the environment runner (rejected; repetitive and error-prone).

- **Recipe names match prompt directory names.** The `save` recipe invokes `docs/prompts/save/README.md`. This coupling is intentional: it creates a predictable mapping that both humans and AI can follow without looking up configuration. Alternative considered: a lookup table or mapping file (rejected; indirection without benefit when the convention is simple enough to be a rule).

- **Settings as a shared import.** Every `.mod.just` imports `settings.just` rather than declaring its own settings. This is DRY applied to configuration. A single file controls shell, dotenv, and the AI backend variable. Alternative considered: per-module settings (rejected; duplicates configuration and creates drift risk).

- **Just installed externally.** Just is installed outside the managed environment. Just is the entry point that invokes the environment manager; managing Just through the environment it bootstraps would create a circular dependency.

## Dependencies

- `Justfile`: root entry point
- `bin/just/settings.just`: shared configuration

## Structure

- `README.md`: Just conventions (the actionable rules)
- `spec/README.md`: this file; reasoning and decisions behind the conventions
