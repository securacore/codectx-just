# Just

Command runner conventions. This document covers the architecture, file organization, and patterns for Just recipes. It does not cover Just syntax; see the [Just manual](https://just.systems/man/en/) for language reference.

For the reasoning behind these conventions, see [spec/README.md](spec/README.md).

## Operational Directive

<rules>

- Just is the sole interface for all repository operations. Do not invoke package managers, build tools, dev servers, linters, or AI tools directly.
- Use the corresponding Just recipe for every task. If no recipe exists for a task, create one before proceeding.
- Just serves as an abstraction layer that decouples workflows from specific tools. The underlying package manager, runtime, or toolchain may change. The Just interface remains stable.

</rules>

## Architecture

The Just configuration is organized into four layers. Each layer has a distinct responsibility.

<rules>

- **Root module** (`bin/just/root/`): Public recipes the developer runs directly (`just build`, `just save`, `just dev`). Imported into the top-level `Justfile` via `import`, so recipes appear without a namespace prefix.
- **AI dispatch layer** (`bin/just/ai/`): Private recipes that route to the active AI backend. The `ai_bin` variable in `settings.just` determines which backend receives the call. Declared as a `mod` in the `Justfile`, so recipes are namespaced (`just ai save`).
- **AI backends** (`bin/just/<provider>/`): Provider-specific implementations. Each backend has a `prompt` recipe that knows how to invoke its AI tool against prompt definitions in `docs/prompts/`. Declared as `mod` in the `Justfile`, so recipes are namespaced (`just <provider> <recipe>`).
- **Settings** (`bin/just/settings.just`): Shared configuration imported by every module. Defines shell, dotenv loading, and the `ai_bin` variable.

</rules>

### Import vs. Mod

<rules>

- Use `import` for the root module. Root recipes appear at the top level of `just -l` with no namespace prefix.
- Use `mod` for all other modules. This namespaces their recipes under the module name and prevents collisions.

</rules>

### Command Flow

Root recipes delegate to the AI dispatch layer. The AI dispatch layer delegates to whichever backend `ai_bin` is set to. The backend executes the provider-specific implementation.

Example flow for `just save`:

1. `bin/just/root/save.just` calls `just ai save`
2. `bin/just/ai/save.just` calls `just {{ai_bin}} save`
3. `bin/just/<provider>/save.just` calls its `prompt` recipe with the appropriate prompt directory

### Syntax Reference

The examples below show the concrete Just syntax for the architecture described above. Provider names and tool invocations are illustrative — the structural patterns are the standard.

#### Root `Justfile`

The root `Justfile` lives at the project root. It declares `mod` for each namespaced module and uses `import` for the root module so its recipes appear at the top level.

```just
mod claude "bin/just/claude/.mod.just"
mod opencode "bin/just/opencode/.mod.just"
mod ai "bin/just/ai/.mod.just"

import "bin/just/root/.mod.just"

# List available commands.
[group('root')]
default:
  just -l
```

#### Module Entry Point (`.mod.just`)

Every module directory has a `.mod.just` file. It imports `settings.just` first, then imports each recipe file in the directory.

```just
# bin/just/root/.mod.just
import "../settings.just"
import "save.just"
import "done.just"
import "build.just"
```

#### Delegation Chain

The following shows the complete delegation chain for a `save` recipe across all four layers.

**Root recipe** (`bin/just/root/save.just`) — public, delegates to the AI dispatch layer:

```just
# AI generated commit.
[group('root')]
save:
  echo "Committing changes, please wait..."
  just ai save
  echo "Done"
```

**AI dispatch recipe** (`bin/just/ai/save.just`) — private, routes to the active backend via `{{ai_bin}}`:

```just
# Invokes AI to generate a commit.
[private]
[group('ai')]
save:
  just {{ai_bin}} save
```

**Backend recipe** (`bin/just/<provider>/save.just`) — private, calls the provider's `prompt` recipe with the prompt directory name and a context string:

```just
# Invokes AI to generate a commit.
[private]
[group('claude')]
save:
  just claude prompt save "Generate a commit, but do NOT push."
```

**Prompt recipe** (`bin/just/<provider>/prompt.just`) — private, the single integration point between Just and the AI CLI. Accepts a `directory` parameter that maps to `docs/prompts/<directory>/README.md` and an optional `context` string:

```just
# Invoke a prompt definition against the AI provider.
[private]
[group('claude')]
prompt directory context="Run the prompt against the entire repo.":
  #!/usr/bin/env bash
  set -euo pipefail

  if ! command -v claude &> /dev/null; then
    echo "Error: This command requires 'claude' to be installed."
    exit 1
  fi

  prompt="{{context}} "
  prompt+="@docs/prompts/{{directory}}/README.md "
  prompt+="Do NOT add AI signatures or co-author attributions."

  claude -p --verbose "$prompt"
```

Each AI provider has its own `prompt.just` with provider-specific CLI invocation. The interface (accepting `directory` and `context`) remains the same across providers.

#### AI Dispatch Composition

The AI dispatch layer can compose multiple operations. For example, a `done` recipe commits via the backend and then pushes:

```just
# Invokes AI to generate a commit and then pushes commit.
[private]
[group('ai')]
done:
  just {{ai_bin}} save
  git push
```

This does not require a corresponding `done` recipe in the backend — the dispatch layer handles the composition directly.

#### Non-AI Recipes

Root recipes that do not involve AI invoke tools directly without going through the dispatch chain:

```just
# Build the application for production.
[group('root')]
build:
  bun run build
```

```just
# Start the development server.
[group('root')]
dev:
  bun run dev
```

These still follow the same conventions: one recipe per file, a descriptive comment, and a `[group()]` annotation.

## File Organization

<rules>

- One recipe per file. The filename matches the recipe name (`save.just` contains the `save` recipe).
- Every module directory has a `.mod.just` file as its entry point. This file imports `settings.just` and all recipe files in the directory.
- `settings.just` lives at `bin/just/settings.just`. Every `.mod.just` imports it.
- All Just files live under `bin/just/`. The only exception is the root `Justfile` at the project root.
- File names use lowercase kebab-case.

</rules>

## Recipe Conventions

<rules>

- Root recipes are public. They are the user-facing interface.
- AI module recipes are private. Mark them with `[private]`.
- Every recipe has a comment on the line above it describing what it does. This comment appears in `just -l` output.
- Every recipe has a `[group()]` annotation. The group name matches the module name.

</rules>

## Shell and Environment

<rules>

- All recipes execute through a managed shell environment configured via `set shell` in `settings.just`.
- Environment variables load from a dotenv file via `set dotenv-filename` in `settings.just`.
- Recipes that need bash scripting use a `#!/usr/bin/env bash` shebang with `set -euo pipefail`.

</rules>

## AI Provider Integration

<rules>

- The `ai_bin` variable in `settings.just` controls which AI backend is active. Set it to the name of the desired backend module.
- AI backends reference prompt definitions in `docs/prompts/`. The recipe name matches the prompt directory name (`save` invokes `docs/prompts/save/README.md`).
- Each backend has a `prompt` recipe that accepts a directory name and optional context string. This is the single point of integration between Just and the AI tool.

</rules>

### Adding a New AI Provider

1. Create a new directory under `bin/just/` named after the provider (e.g., `bin/just/newprovider/`).
2. Create a `.mod.just` that imports `settings.just` and the recipe files.
3. Create a `prompt.just` that invokes the provider's CLI with the prompt file from `docs/prompts/`.
4. Create recipe files for each AI recipe that call the `prompt` recipe.
5. Add a `mod` declaration in the root `Justfile`.
6. Set `ai_bin` in `settings.just` to the new provider name to make it the default.

### Adding a New AI Recipe

1. Create the prompt definition in `docs/prompts/<name>/README.md`.
2. Create `<name>.just` in each AI backend directory, calling the `prompt` recipe with the prompt directory name.
3. Create `<name>.just` in `bin/just/ai/` to dispatch to `just {{ai_bin}} <name>`.
4. Import the new file in each `.mod.just`.
5. Optionally create a root recipe in `bin/just/root/` if the command is user-facing.

## Configuration Reference

All runtime configuration is defined in `bin/just/settings.just`. When this document and configuration conflict, configuration wins.

### Initial `settings.just` Setup

When initializing Just support in a project, create `bin/just/settings.just` with the following structure:

```just
# Suppress command echo in output.
set quiet

# Load environment variables from a dotenv file.
set dotenv-load
set dotenv-override
set dotenv-filename := ".env.development"

# Execute all recipes through a managed shell environment.
# Replace the command below with the appropriate environment runner for the project
# (e.g., devbox, nix, mise, or any tool that provides a consistent shell).
set shell := ["devbox", "run", "-q", "--"]

# Name of the active AI backend. Must match the directory name of a
# backend module under bin/just/ (e.g., "claude", "opencode").
ai_bin := "claude"
```

Adapt the `set shell` value to whatever environment manager the project uses. The rest of the settings should remain as shown unless there is a specific reason to diverge.
