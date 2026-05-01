# Harp Skill for Claude Code

A skill that teaches Claude to author and maintain [Harp.sh](https://harp.sh) static sites — layouts, partials, `_data.json` metadata, `_harp.json` configuration, and the EJS/Markdown asset pipeline.

## What it provides

When invoked, the skill gives Claude:

- Project structure conventions: served root, the underscore-prefix rule, directory-as-routing.
- How to compose `_layout.ejs` files with `<%- yield %>` and `<%- partial(...) %>`.
- How to define and iterate over `_data.json`, including the precedence between `partial()` arguments, the page's own metadata, and `_harp.json` globals.
- The current asset pipeline — which source extensions compile to which output:
  - `.ejs`, `.jade`, `.md` → `.html`
  - `.sass`, `.scss`, `.less`, `.styl` → `.css`
  - `.coffee`, `.cjs`, `.jsx` → `.js`
- The decision rule for choosing `.ejs` vs `.md` for a new page.
- The `_harp.json` schema (`globals`, `basicAuth`, `$ENV_VAR` substitution, runtime `environment` flag).
- The CLI form: `harp <source>` to serve, `harp <source> <build>` to compile.
- Patterns for refactoring duplicated chrome into layouts and partials.

## When it triggers

Claude consults the skill when it detects a Harp project — `_layout.ejs`, `_data.json`, `_harp.json`, `_partials/`, or a legacy `harp.json` paired with a `public/` directory — or when a user asks for help with markup, layouts/partials, globals, content iteration, or scaffolding in such a project. Explicit mentions of Harp aren't required.

## Installation

Place the `harp/` directory inside one of Claude Code's skill search paths.

**Available globally** (all projects):

```
mkdir -p ~/.claude/skills
ln -s "$(pwd)/harp" ~/.claude/skills/harp
```

**Available in a single project:**

```
mkdir -p <project>/.claude/skills
ln -s "$(realpath harp)" <project>/.claude/skills/harp
```

Confirm the skill is loaded by listing available skills inside Claude Code.

## Repository layout

```
.
├── harp/
│   ├── SKILL.md              # the skill prompt
│   └── evals/
│       ├── evals.json        # test prompts and fixture references
│       └── inputs/           # input fixtures referenced by evals.json
├── harp-workspace/           # generated — gitignored
├── .gitignore
└── README.md
```

## Development

The skill is iterated against a small set of test prompts in `harp/evals/evals.json`. Each entry has a prompt, an expected-output description, and a list of input file paths (relative to `harp/`).

A typical iteration:

1. Spawn paired subagents for each test case — one with access to the skill, one baseline.
2. Compare outputs side-by-side in the eval viewer; capture qualitative feedback.
3. Apply targeted edits to `harp/SKILL.md` based on the feedback.
4. Re-run the evals and compare against the previous iteration.

Generated artifacts land under `harp-workspace/iteration-N/` and are excluded from version control.

## Compatibility

The skill targets current Harp and Terraform behavior:

- [`harp`](https://github.com/sintaxi/harp) 0.47.x — server, CLI, `_harp.json` parsing, basic auth, environment substitution.
- [`terraform`](https://github.com/sintaxi/terraform) 1.23.x — the rendering engine: layouts, partials, `_data.json`, asset pipeline.

Older Harp projects using the `harp.json` + `public/` shape are still supported — the skill mirrors whichever shape the existing project uses.
