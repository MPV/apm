# New package manager questionnaire

Did you read our documentation on adding a package manager?

- [x] I've read the [adding a package manager](adding-a-package-manager.md) documentation.

## Basics

### What's the name of the package manager?

**APM** (Agent Package Manager) — https://github.com/microsoft/apm

### What language(s) does this package manager support?

APM is **language-agnostic**. It manages AI agent context (instructions, prompts, skills, hooks, MCP server declarations) for any project, regardless of the project's programming language. The tool itself is written in Python and distributed via PyPI (`apm-cli`).

The closest analogy is how EditorConfig or Prettier manage editor/formatter configuration regardless of project language — APM does the same for AI coding-agent configuration.

### How popular is this package manager?

- **~2 000 GitHub stars** (as of April 2026) and growing rapidly since its release in late 2025. See [live count](https://github.com/microsoft/apm).
- Published by **Microsoft** as an open-source project.
- Targets the fast-growing AI coding-agent ecosystem (GitHub Copilot, Claude Code, Cursor, OpenCode, Codex).
- Installable via `pip install apm-cli`, Homebrew, and platform-specific installers.

### Does this language have other (competing?) package managers?

- [x] Yes (give names).
- [ ] No.

There is no other widely adopted package manager for AI agent context. The space is new. Some adjacent tools exist but serve different purposes:

- **Rules files / CLAUDE.md / .cursorrules** — manual, per-project configuration files; no dependency resolution, versioning, or sharing.
- **Copilot Extensions / MCP registries** — provide runtime tool servers, not declarative dependency management for agent instructions and skills.

APM is the first tool to treat agent context as versioned, shareable, lockable packages with transitive dependency resolution.

### What are the big selling points for this package manager?

- **Reproducible agent context**: lock file (`apm.lock.yaml`) pins every dependency to an exact git commit SHA, ensuring identical setups across machines.
- **Multi-agent-runtime support**: a single `apm.yml` compiles output for GitHub Copilot, Claude Code, Cursor, OpenCode, and Codex simultaneously.
- **Git-native, registry-optional**: packages are ordinary git repos (GitHub, GitLab, Bitbucket, Azure DevOps, self-hosted). No centralised registry is required.
- **Transitive dependency resolution**: full BFS dependency graph with circular-dependency detection, conflict resolution, and depth tracking.
- **Enterprise governance**: built-in policy engine (`apm-policy.yml`), supply-chain security auditing (`apm audit`), and content-hash verification.

## Detecting package files

### What kind of package files, and names, does this package manager use?

| File | Format | Purpose |
|------|--------|---------|
| `apm.yml` | YAML 1.2 | Package manifest (the equivalent of `package.json`) |
| `apm.lock.yaml` | YAML 1.2 | Lock file (the equivalent of `package-lock.json`) |

Both files live at the project root. The manifest filename is always exactly `apm.yml` — there is no alternate naming convention.

### Which [`managerFilePatterns`](../usage/configuration-options.md#managerfilepatterns) pattern(s) should Renovate use?

```
["(^|/)apm\\.yml$"]
```

Note: the lock file `apm.lock.yaml` lives alongside `apm.yml` and should be treated as an artifact that Renovate updates (see [Artifacts](#artifacts)), but does not need to be in `managerFilePatterns` itself.

### Do many users need to extend the [`managerFilePatterns`](../usage/configuration-options.md#managerfilepatterns) pattern for custom file names?

- [ ] Yes, provide details.
- [x] No.

The manifest filename is always `apm.yml`. The specification does not allow alternate names.

### Is the [`managerFilePatterns`](../usage/configuration-options.md#managerfilepatterns) pattern going to get many "false hits" for files that have nothing to do with package management?

No. While `apm.yml` is a short name, its internal structure is distinctive (requires `name` and `version` top-level keys, and uses an `apm`/`mcp`-keyed `dependencies` block). A quick structural check (presence of `dependencies.apm` or `dependencies.mcp`) eliminates false positives.

## Parsing and Extraction

### Can package files have "local" links to each other that need to be resolved?

No. Each `apm.yml` is self-contained. Dependencies point to external git repositories (or local filesystem paths for development), not to other `apm.yml` files within the same repository.

### Package file parsing method

The package files should be:

- [ ] Parsed together (in serial).
- [x] Parsed independently.

### Which format/syntax does the package file use?

- [ ] JSON
- [ ] TOML
- [x] YAML
- [ ] Custom (explain below)

Standard YAML 1.2.

### How should we parse the package files?

- [x] Off the shelf parser.
- [ ] Using regex.
- [ ] Custom-parsed line by line.
- [ ] Other.

A standard YAML parser is sufficient. Dependencies live under well-defined keys (`dependencies.apm`, `dependencies.mcp`, `devDependencies.apm`, `devDependencies.mcp`).

### Does the package file have different "types" of dependencies?

- [x] Yes, production and development dependencies.
- [ ] No, all dependencies are treated the same.

APM has two axes of categorisation:

1. **Production vs development**: `dependencies` vs `devDependencies` (dev deps are excluded from `apm pack --format plugin` bundles).
2. **APM vs MCP**: each group has an `apm` list (git-hosted agent-context packages) and an `mcp` list (MCP server declarations).

For Renovate's purposes, the most relevant entries are under `dependencies.apm` and `devDependencies.apm` — these reference versioned git repositories.

### List all the sources/syntaxes of dependencies that can be extracted

**APM dependencies** (`dependencies.apm` / `devDependencies.apm`) come in two forms:

**1. String form:**

```yaml
dependencies:
  apm:
    # GitHub shorthand (default host)
    - microsoft/apm-sample-package              # latest default branch
    - microsoft/apm-sample-package#v1.0.0       # pinned to tag
    - microsoft/apm-sample-package#main         # branch ref

    # Non-GitHub hosts
    - gitlab.com/acme/coding-standards
    - bitbucket.org/team/repo#main

    # Full URLs
    - https://github.com/microsoft/apm-sample-package.git#v2.0.0
    - git@github.com:microsoft/apm-sample-package.git

    # Virtual packages (subdirectories/files within a repo)
    - ComposioHQ/awesome-claude-skills/brand-guidelines
    - contoso/prompts/review.prompt.md

    # Azure DevOps
    - dev.azure.com/org/project/_git/repo

    # Local path (development only — not relevant for Renovate)
    - ./packages/my-shared-skills
```

**2. Object form:**

```yaml
dependencies:
  apm:
    - git: https://gitlab.com/acme/repo.git
      path: instructions/security
      ref: v2.0
      alias: acme-sec
```

In object form, `ref` carries the version (equivalent to the `#ref` fragment in string form). Renovate should update the `ref` value the same way it updates the `#v1.0.0` fragment in string-form dependencies (e.g. `ref: v2.0` → `ref: v2.1.0`).

**MCP dependencies** (`dependencies.mcp` / `devDependencies.mcp`) reference MCP servers. These are either registry-backed identifiers or self-defined server configs. They typically do not carry semver versions and are less relevant for Renovate initially.

### Describe which types of dependencies above are supported and which will be implemented in future

**Initial scope (recommended):**

- `dependencies.apm` and `devDependencies.apm` entries that reference git repositories with a tag/version ref (e.g. `owner/repo#v1.0.0`). These have clear version semantics and are the primary use case for automated updates.

**Future scope:**

- `dependencies.mcp` — MCP server registry references. These may gain versioning as the MCP ecosystem matures.
- Dependencies pinned to branch names or commit SHAs — these could be updated to the latest commit on that branch, though the value proposition is lower.

## Versioning

### What versioning scheme does the package file(s) use?

**Semantic Versioning** (semver). The `version` field in `apm.yml` must match `^\d+\.\d+\.\d+` with optional pre-release/build suffixes (e.g. `1.0.0`, `2.1.0-beta`, `1.0.0+build123`).

Dependency refs typically use git tags prefixed with `v` (e.g. `#v1.0.0`), matched by `^v?\d+\.\d+\.\d+`.

### Does this versioning scheme support range constraints, like `^1.0.0` or `1.x`?

- [ ] Supports range constraints (for example: `^1.0.0` or `1.x`), provide details.
- [x] No.

APM does **not** support version ranges. Dependencies pin to an exact ref (tag, branch, or commit SHA). When no ref is specified, APM resolves to the latest default-branch commit and pins the exact SHA in the lock file.

This means Renovate would update a dependency by changing the ref value (e.g. `#v1.0.0` → `#v1.1.0`) and letting `apm install` regenerate the lock file.

## Lookup

### Is a new datasource required?

- [ ] Yes, provide details.
- [x] No.

APM dependencies are git repositories hosted on GitHub, GitLab, Bitbucket, Azure DevOps, etc. Renovate's existing **git-tags** datasource should work for tag-based version refs. For GitHub-hosted packages (the vast majority), the **github-tags** or **github-releases** datasource applies directly.

### Will users want (or need to) set a custom host or custom registry for Renovate's lookup?

- [x] Yes, provide details.
- [ ] No.

APM supports any git-accessible host (GitHub, GitLab, Bitbucket, Azure DevOps, self-hosted instances). Users with dependencies on non-GitHub hosts will need Renovate to look up tags on the correct host.

Where can Renovate find the custom host/registry?

- [ ] No custom host or registry is needed.
- [x] In the package file(s), provide details.
- [ ] In some other file inside the repository, provide details.
- [ ] User needs to configure Renovate where to find the information, provide details.

The host is encoded directly in the dependency string:
- `gitlab.com/acme/coding-standards#v1.0.0` → host is `gitlab.com`
- `bitbucket.org/team/repo#v2.0.0` → host is `bitbucket.org`
- `microsoft/apm-sample-package#v1.0.0` → implicit host is `github.com`

### Are there any constraints in the package files that Renovate should use in the lookup procedure?

- [ ] Yes, there are constraints on the parent language (for example: supports only Python `v3.x`), provide details.
- [ ] Yes, there are constraints on the parent platform (for example: only supports Linux, Windows, etc.), provide details.
- [ ] Yes, some other kind of constraint, provide details.
- [x] No constraints.

### Will users need the ability to configure language or other constraints using Renovate config?

- [ ] Yes, provide details.
- [x] No.

## Artifacts

### Does the package manager use a lock file or checksum file?

- [x] Yes, uses lock file.
- [ ] Yes, uses checksum file.
- [ ] Yes, uses lock file _and_ checksum file.
- [ ] No lock file or checksum.

APM uses `apm.lock.yaml` (YAML 1.2). It records every dependency's exact commit SHA, deployed file paths, and content hashes. The lock file also tracks SHA-256 hashes of deployed files for integrity verification, but these are embedded in the lock file itself — there is no separate checksum file.

### Is the locksum or checksum mandatory?

- [ ] Yes, locksum is mandatory.
- [ ] Yes, checksum is mandatory.
- [ ] Yes, lock file _and_ checksum are mandatory.
- [x] No mandatory locksum or checksum.
- [ ] Package manager does not use locksums or checksums.

The lock file is strongly recommended and should be committed to version control, but `apm install` works without a pre-existing lock file (it creates one). Enterprise policy rules can enforce lock file presence via `apm audit`.

### If lockfiles or checksums are used: what tool and exact commands should Renovate use to update one (or more) package versions in a dependency file?

**Tooling prerequisite:** Renovate needs `apm-cli` available in the update environment. Install via:

```bash
pip install apm-cli
```

Then, after Renovate modifies the version ref in `apm.yml`:

```bash
apm install
```

This re-resolves affected dependencies and updates `apm.lock.yaml`. For a full re-resolve of all dependencies:

```bash
apm install --update
```

Note: `apm install` writes cloned repositories to `apm_modules/` (typically gitignored). Renovate should only commit changes to `apm.yml` and `apm.lock.yaml`, not the `apm_modules/` directory.

### Package manager cache

#### Does the package manager use a cache?

- [x] Yes, provide details.
- [ ] No.

APM caches cloned git repositories in `apm_modules/` within the project directory. On subsequent installs, it reuses cached repos and verifies content hashes from the lock file.

#### If the package manager uses a cache, how can Renovate control the cache?

- [ ] Package manager does not use a cache.
- [x] Controlled via command line interface, provide details.
- [ ] Controlled via environment variables, provide details.

Running `apm install --update` forces a full re-resolve, bypassing cached commit SHAs. The `apm_modules/` directory can also simply be deleted before install for a clean state.

#### Should Renovate keep a cache?

- [ ] Yes, ignore/disable the cache.
- [x] No.

No special cache handling needed. Renovate modifies `apm.yml`, then runs `apm install` which handles caching internally.

### Generating a lockfile from scratch

Renovate can perform "lock file maintenance" by getting the package manager to generate a lockfile from scratch.
Can the package manager generate a lockfile from scratch?

- [x] Yes, explain which command Renovate should use to generate the lockfile.
- [ ] No, the package manager does _not_ generate a lockfile from scratch.
- [ ] No, the package manager does not use lockfiles.

Delete the existing lock file and run:

```bash
rm -f apm.lock.yaml && apm install
```

This resolves all dependencies fresh and produces a new `apm.lock.yaml`.

## Other

### What else should we know about this package manager?

1. **Git-tag-based versioning only** — APM does not use a centralised package registry with its own version index. Versions are git tags. This aligns well with Renovate's existing git-tags datasource.

2. **Virtual packages** — a single dependency string can target a subdirectory or file within a larger monorepo (e.g. `ComposioHQ/awesome-claude-skills/brand-guidelines`). The version tag applies to the whole repository, not the subdirectory. Renovate should treat the base `owner/repo` as the versioned unit.

3. **MCP dependencies are a separate concern** — `dependencies.mcp` entries reference MCP (Model Context Protocol) servers. These are structurally different from APM deps and are not yet versioned in a way that lends itself to automated updates. They can be deferred to a future iteration.

4. **Enterprise policy** — organisations can define `apm-policy.yml` files that restrict allowed dependency sources, enforce lock file presence, and require content-hash verification. Renovate-generated PRs would need to pass these policy checks, but that happens naturally via `apm install` and CI.

5. **Growing ecosystem** — APM is under active development by Microsoft. The manifest schema and lock file format are versioned (`lockfile_version: "1"`) and designed for forward compatibility. The project has ~2 000 GitHub stars as of April 2026 and is the recommended way to manage agent context for GitHub Copilot Coding Agent.
