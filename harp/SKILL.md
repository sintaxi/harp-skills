---
name: harp
description: Author and modify Harp.sh static sites — Harp is a zero-config static web server, generator, and bundler that uses EJS, Markdown, `_layout` files, partials, `_data.json` metadata, and an optional `_harp.json` for site-wide globals. Use this skill whenever you see `_layout.ejs`/`_layout.jade`, `_data.json`, `_partials/`, or a `_harp.json` file in a project, or when the user asks to create/change markup in such a project, scaffold or refactor layouts and partials, set up globals or per-directory data, iterate over JSON data inside templates, configure 404/200 fallback pages, or run `harp` to serve or compile — apply it even if the user doesn't explicitly say "Harp."
---

# Harp

Harp is a zero-config static web server, generator, and bundler. It's a thin server/CLI layer (the `harp` package) wrapping Terraform (the asset rendering engine). Point Harp at a folder and it serves the folder. Files with template extensions are compiled on the fly; everything else is served as static files. Files and directories whose names start with an underscore (`_layout.ejs`, `_partials/`, `_data.json`) are never served — they're for templating and metadata.

The reason these conventions exist: Harp wants the directory tree itself to be the routing and structure of the site. No build config, no manifest, no plugin registry. The filesystem is the source of truth.

## Detecting a Harp project

Before scaffolding or restructuring, check whether you're already inside a Harp project. Strong indicators:

- `_layout.ejs`, `_layout.jade`, or any `_data.json` file in the tree
- `_harp.json` in the source directory (optional config file)
- `harp` listed in `package.json` dependencies
- Legacy: a `harp.json` file with a sibling `public/` subdirectory full of templates (older app-style — see "Project shape")

If none of these exist, the directory is not a Harp project yet — confirm with the user before scaffolding.

## The five rules

1. **Convention over configuration.** A single `index.ejs` (or `index.md`) is enough to have a working site.
2. **The directory you point `harp` at is the served root.** No config file is required. If you want site-wide globals, HTTP Basic Auth, or environment-variable substitution, drop a `_harp.json` directly into that directory — the underscore prefix keeps it from being served. (A legacy mode exists where `harp.json` at a parent dir maps to a `public/` subfolder as the served root — see "Project shape" — but new projects don't need it.)
3. **Underscore-prefixed names are never served.** This is the convention for layouts, partials, and metadata. The middleware blocks them; the compiler skips them.
4. **The asset pipeline is automatic.** Source files get compiled to standard web output:
   - `.ejs`, `.md`, `.jade` → `.html`
   - `.sass`, `.scss`, `.less`, `.styl` → `.css`
   - `.coffee`, `.cjs`, `.jsx` → `.js` (the last two via esbuild)
   To control output type explicitly, use a double extension: `feed.xml.ejs` is served as `feed.xml`. Files with non-source extensions (images, fonts, `.html`, video, etc.) are served and copied through unchanged.
5. **Metadata lives in `_data.json` files.** Per-directory JSON injects scoped variables into the templates next to it.

Direct requests to source files are blocked. `/about.md` returns 404; only the rendered URL `/about` works.

## Choosing the file extension

For each new page, pick the extension that fits the content:

- **`.md`** — content pages: prose, articles, blog posts, longform copy, anything that's mostly text.
- **`.ejs`** — layouts, partials, structural pages, index/list pages, anything needing loops, conditionals, or `partial()` calls.
- **`.jade`** — fully supported by the engine. Use only if the existing project already uses Jade. New examples in this skill use EJS.
- **`.html`** — pure static HTML pages with no templating need; Harp serves them unchanged.

Decision rule: if the existing project leans one way (look at neighboring files in the same directory), match it. Otherwise default to `.md` for content, `.ejs` for structure.

## Project shape

The directory you point `harp` at is the served root. Name it whatever you want — `public/`, `src/`, `mysite/`, or just `.`. The directory name doesn't carry meaning to Harp; only the contents do.

The simplest possible Harp project is a folder of template files:

```
public/
├── _layout.ejs
├── index.ejs
└── about.md
```

Run with `harp ./public`. No config file required.

Add a `_harp.json` to that same directory if you want **site-wide globals**, **HTTP Basic Auth**, or **`$ENV_VAR` substitution**:

```
public/
├── _harp.json
├── _layout.ejs
├── _data.json
├── index.ejs
├── about.md
├── _partials/
│   ├── _header.ejs
│   └── _footer.ejs
└── articles/
    ├── _data.json
    ├── _layout.ejs        # overrides parent layout for this subtree
    ├── hello-world.md
    └── hello-brazil.md
```

The underscore prefix on `_harp.json` keeps Harp from serving it as a page. Per-directory `_data.json` files work the same way regardless of whether `_harp.json` is present.

**Legacy app-style** (`harp.json` + `public/` subdirectory) — older Harp projects place `harp.json` in a parent directory and put all the source files inside a `public/` subfolder:

```
project-root/
├── harp.json
└── public/
    ├── _layout.ejs
    ├── index.ejs
    └── about.md
```

You serve this with `harp ./project-root` and Harp resolves the served root to `./project-root/public/`. If you walk into a project shaped this way, mirror it; don't restructure unless the user asks. For new sites, point `harp` at the source directory directly and use `_harp.json` if you need config.

## Layouts

A layout is an EJS (or Jade) template that wraps page content. Conventions:

- Name it `_layout.ejs` (or `_layout.jade`).
- Place it in the directory whose pages it should wrap.
- A subdirectory's `_layout.ejs` overrides the parent's for pages in that subtree.
- Mark the content insertion point with `<%- yield %>` — the dash matters; it outputs the wrapped page content unescaped. (`yield` is only available *inside* the layout, not the page being wrapped.)

Example `_layout.ejs`:

```ejs
<!doctype html>
<html>
  <head>
    <title><%= title %> · <%= siteTitle %></title>
  </head>
  <body>
    <%- partial("_partials/_header") %>
    <main><%- yield %></main>
    <%- partial("_partials/_footer") %>
  </body>
</html>
```

Per-page opt-out: in the page's `_data.json` entry, set `"layout": false`.

Per-page alternate layout: set `"layout": "path/to/_alt-layout"`. The path resolves relative to the page's own directory first, then falls back to the project root. So both `"layout": "_special-layout"` (sibling) and `"layout": "../../_layout"` (parent traversal) work.

Two automatic exemptions:
- Pages that emit a `.json` extension (e.g., `feed.json.ejs`) skip the layout automatically — useful for API endpoints and feeds.
- The `layout` key in `_data.json` is not inherited by partials, so partials never accidentally get wrapped.

## Partials

Partials are reusable template fragments. They:

- Have filenames prefixed with `_` so they aren't served as pages.
- Are included with the global `partial()` function.
- Optionally accept a data object passed as the second argument.
- Are cross-language: an EJS template can include a Jade partial and vice versa, and either can include a `.md` file as a partial.
- **Cannot** be called from inside a `.md` body. The reason: Markdown is compiled to static HTML by `marked`, which doesn't execute template code. Only `.ejs` and `.jade` execute template logic. If you need partials inside markdown content, either convert the file to `.ejs`, or include the `.md` as a partial from an `.ejs` page.

EJS usage:

```ejs
<%- partial("_partials/_header") %>
<%- partial("_partials/_card", { title: "Hello", body: "World" }) %>
<%- partial("articles/hello-world") %>   <!-- include an .md content file -->
```

Paths are relative to the served root (the directory you point `harp` at), and the file extension is omitted.

The number-one EJS mistake: writing `<%= partial(...) %>` instead of `<%- partial(...) %>`. The first form HTML-escapes the partial's output, so the user sees literal `&lt;div&gt;` markup. Same rule for `yield`.

## `_harp.json` — site config

`_harp.json` is a flat object placed directly in the source directory. It is **not** a `{ "globals": { ... } }` wrapper around everything; `globals` is one of several recognized top-level keys. (Legacy app-style projects use the same schema in a `harp.json` at a parent directory — see "Project shape" above.)

Recognized top-level keys:

| Key | What it does |
|-----|--------------|
| `globals` | Object of site-wide template variables, available as top-level vars in every template. |
| `basicAuth` | HTTP Basic Auth credentials. Value is `"user:pass"` or `["user:pass", "alice:s3cret"]`. |
| Any other key | Preserved and emitted into the compiled `globals.json`. Treat as your own config namespace. |

(The same set applies in a legacy `harp.json`.)

Plus one runtime-injected addition: `globals.environment` is automatically set to the `NODE_ENV` value (`"development"` by default, `"production"` when `NODE_ENV=production`). Templates can branch on it directly:

```ejs
<% if (environment === "production") { %>
  <script src="/analytics.js"></script>
<% } %>
```

Example:

```json
{
  "globals": {
    "siteTitle": "Acme",
    "author": "Brock",
    "siteUrl": "https://example.com"
  },
  "basicAuth": ["preview:letmein"]
}
```

**Environment variable substitution.** Strings of the form `"$VAR_NAME"` anywhere in `_harp.json` are replaced at load time with `process.env.VAR_NAME`. Useful for secrets and per-environment values:

```json
{
  "globals": {
    "stripeKey": "$STRIPE_PUBLISHABLE_KEY",
    "apiBase": "$API_BASE"
  }
}
```

There is **no** `environment.development` / `environment.production` nested override structure — that's an old-doc artifact. Use either the runtime `environment` variable or `$ENV_VAR` substitution.

Use `globals` for things that are genuinely site-wide and rarely change between pages: site name, default author, base URL, social handles, feature flags.

## Per-directory data — `_data.json`

`_data.json` provides metadata scoped to the files alongside it. Keys match filenames (without extension); values are objects whose properties become variables inside the matching template.

`articles/_data.json`:

```json
{
  "hello-world": {
    "title": "Hello World",
    "date": "2026-01-12",
    "author": "Jane",
    "tags": ["intro", "meta"]
  },
  "hello-brazil": {
    "title": "Hello Brazil",
    "date": "2026-03-04",
    "author": "Jane",
    "tags": ["travel"]
  }
}
```

Inside `articles/hello-world.ejs`, `<%= title %>` resolves to `"Hello World"`. The same applies to `.jade`. For `.md` pages, see "Markdown handling" below — the data is attached to the page's render context (and reachable from the wrapping layout) but the markdown body itself can't interpolate variables.

The whole map is also reachable from any template as `public.articles._data`. The `public` namespace always wraps the served-root data tree regardless of whether your project has a literal `public/` directory or not — that's how an index page enumerates the entries.

**Variable precedence**, from highest to lowest:

1. Data passed explicitly into `partial(path, { … })`
2. The page's own entry in `_data.json`
3. Globals from `_harp.json`

Keys that aren't defined for a given filename are simply `undefined` in that template — guard with `typeof title !== 'undefined'` or `<% if (typeof title !== 'undefined') { %>` if you need defaults. The `layout` key is special and not inherited by partials.

## Iterating over data

The bread-and-butter Harp pattern: a list/index page renders an index of items defined in a sibling directory's `_data.json`.

```ejs
<!-- articles/index.ejs -->
<h1>Articles</h1>
<ul>
  <% for (var slug in public.articles._data) { %>
    <% var article = public.articles._data[slug] %>
    <li>
      <a href="/articles/<%= slug %>">
        <strong><%= article.title %></strong>
        <span><%= article.date %></span>
      </a>
    </li>
  <% } %>
</ul>
```

To control ordering (JSON object key order is not reliable for display), pull the keys, sort them, then loop:

```ejs
<%
var slugs = Object.keys(public.articles._data).sort(function(a, b) {
  return public.articles._data[b].date.localeCompare(public.articles._data[a].date)
})
%>
<% slugs.forEach(function(slug) { %>
  <% var article = public.articles._data[slug] %>
  <a href="/articles/<%= slug %>"><%= article.title %></a>
<% }) %>
```

To list actual files in a directory (assets, not metadata) — image galleries, downloads — use `_contents`:

```ejs
<% public.images._contents.forEach(function(file) { %>
  <% if (/\.(jpg|png|gif|webp)$/i.test(file)) { %>
    <img src="/images/<%= file %>" alt="" />
  <% } %>
<% }) %>
```

`_contents` is an array of filename strings only. Files starting with `_` or `.` are excluded, and **nested subdirectories are not included** — only files in that exact directory.

## EJS syntax cheat sheet

| Tag | Purpose |
|-----|---------|
| `<% code %>` | Run JavaScript, produce no output |
| `<%= value %>` | Output, **HTML-escaped** (use for nearly all dynamic values) |
| `<%- value %>` | Output, **unescaped** (use for `yield`, `partial()`, and trusted HTML) |
| `<%# comment %>` | Comment, not rendered |

Conditionals and loops are plain JavaScript:

```ejs
<% if (current.source === "index") { %>
  <h1>Welcome</h1>
<% } else { %>
  <h1><%= title %></h1>
<% } %>

<ul>
  <% (tags || []).forEach(function(tag) { %>
    <li><%= tag %></li>
  <% }) %>
</ul>
```

## The `current` object — active state for navigation

`current` is automatically populated for every page render and describes where you are.

- `current.path` — array of path segments without extensions. Visiting `/articles/hello-world` gives `["articles", "hello-world"]`. Visiting `/articles/` gives `["articles", "index"]`.
- `current.source` — the last element of `path`.
- For double-extension files like `feed.json.ejs`, the extension portion stays: `current.source === "feed.json"`.

Typical use in a nav partial:

```ejs
<nav>
  <a href="/"          class="<%= current.path.length === 1 && current.source === 'index' ? 'active' : '' %>">Home</a>
  <a href="/about"     class="<%= current.source === 'about' ? 'active' : '' %>">About</a>
  <a href="/articles"  class="<%= current.path[0] === 'articles' ? 'active' : '' %>">Articles</a>
</nav>
```

The same partial can be used everywhere — `current` makes it self-aware.

## Markdown handling

`.md` files compile to HTML and support GitHub Flavored Markdown, including fenced code blocks (```` ```js ```` produces `<code class="language-js">…</code>`, ready for Prism / Highlight.js).

Markdown files:

- Are wrapped by the nearest parent `_layout.ejs` like any other page.
- Have their `_data.json` entry attached to the render context — but the **markdown processor doesn't substitute variables**. `<%= title %>` inside a `.md` body appears as literal text. The values are reachable from the wrapping layout (which is a real EJS template) and from any other template via `public.articles._data`, which is the usual way you'd use them.
- **Cannot** call `partial()`, use `<% if %>`/`<% for %>`, or interpolate variables inside the body — the markdown processor (`marked`) only renders HTML, it doesn't execute template code. Only `.ejs` and `.jade` execute template logic.

A common workflow: keep prose in `.md` (so authors can write plain Markdown), and put all the chrome — `<title>` from `_data.json`, navigation, related-post lists, etc. — in the `_layout.ejs` that wraps the page. If you need template logic inside the body, write the page as `.ejs` instead.

## Stylesheets and JavaScript

Harp's pipeline isn't only for HTML. Source files in supported preprocessor extensions are compiled in place:

- **CSS:** `main.scss`, `main.sass`, `main.less`, or `main.styl` all serve at `/main.css` and are emitted as `main.css` on compile.
- **JavaScript:** `app.coffee`, `app.cjs`, or `app.jsx` all serve at `/app.js`. `.cjs` and `.jsx` go through esbuild, so JSX and modern JS work without configuration.

Reference the compiled URL in templates (`<link rel="stylesheet" href="/main.css">`), not the source extension. Direct requests to source files (e.g., `/main.scss`) are blocked.

## Custom 200 / 404 pages

A `404.html` file (or any compilable equivalent like `404.ejs`) anywhere in the tree is used as the not-found page. The lookup cascades up the directory hierarchy: a request inside `/articles/` will try `articles/404.html` first, then a project-root `404.html`.

A `200.html` works the same way for catch-all rendering — useful for SPAs where every unmatched URL should render a single shell page that the client-side router takes over.

## CLI

Harp is installed globally:

```
sudo npm install -g harp
```

Two invocations cover the workflow:

**Serve locally:**

```
harp <source>
```

Defaults: port `9000`, host `0.0.0.0`. Common flags:

- `-p, --port <port>`
- `-h, --host <host>`
- `-s, --silent`

The dev server has no live reload — refresh the browser to see changes.

Examples:
```
harp ./public                  # serve a folder named "public"
harp ./mysite --port 3000      # serve a folder named "mysite" on port 3000
harp .                         # serve current directory
NODE_ENV=production harp .     # forces globals.environment === "production"
```

The directory name is up to you — `public/`, `src/`, `mysite/`, anything. Harp serves whatever directory you give it; just point at the directory that contains your templates (and optionally `_harp.json`).

**Compile to static assets:**

```
harp <source> <build>
```

Default output directory is `www/` (relative to the source) when `<build>` is omitted in some embedded uses. Examples: `harp ./public ./www` produces a flat directory of HTML/CSS/JS deployable to any static host (S3, Netlify, GitHub Pages, Cloudflare Pages).

What's emitted:
- All template sources rendered to their compiled extensions.
- All non-source static files (images, fonts, `.html`, video, etc.) copied through unchanged.
- `_layout.*`, `_partials/*`, `_data.json`, and `_harp.json` (or legacy `harp.json`) are **not** emitted — they exist only at build time.
- A `globals.json` is written for the runtime helpers; safe to ignore unless you're embedding Harp.

## Refactoring an existing page into layouts and partials

When asked to "break a file into layouts and partials":

1. **Read the file** and separate page-specific content from shared chrome (doctype, `<head>`, header, footer, body wrapper).
2. **Move shared chrome into `_layout.ejs`.** Replace the page-specific region with `<%- yield %>`. The page file now contains only its own content.
3. **Pull repeated chunks into partials.** Anything that appears in more than one place — a card, a hero, a CTA, a nav — goes into `_partials/_name.ejs`. Parameterize what varies via the data object argument to `partial()`.
4. **Lift shared data into JSON.** Hardcoded lists of items (articles, products, team members) belong in `_data.json` (per directory) or `_harp.json` globals (site-wide). Then iterate in the template.
5. **Verify the served URL didn't change.** Don't rename the page file as part of refactoring unless the user asked to.

## Reorganizing files

URL ↔ path mapping:

- `foo/bar.md` → `/foo/bar` (and `/foo/bar.html`)
- `foo/index.md` → `/foo` and `/foo/`
- Renaming a page changes its URL. Call this out and offer to update internal links and `_data.json` keys.
- Moving a page into a subdirectory may bring it under a different `_layout.ejs` — verify the new layout still works for that page, and check whether `_data.json` needs to follow.

## Common pitfalls

- Using `<%= yield %>` or `<%= partial(...) %>` (escaped) instead of `<%- … %>` (unescaped) — produces visible markup tags on the page.
- Pointing `harp` at the parent of your source directory — you'll see a directory listing or 404s on every URL. Run `harp` against the directory that contains your templates directly.
- Naming a layout `layout.ejs` (no underscore) — it gets served as a page rather than treated as a layout.
- Calling `partial()` inside a `.md` file — silently does nothing; convert to `.ejs` or invert the inclusion.
- `_data.json` keys must match filenames **without** the extension. `"hello-world.md"` as a key won't bind.
- Wrapping everything in `_harp.json` under `"globals"`. `globals` is one key among several at the top level (`basicAuth` is a sibling, not a child of `globals`).
- Referencing `environment.development` / `environment.production` in `_harp.json` — that nested structure doesn't exist. Use the runtime `environment` variable instead.
- JSON object key order isn't guaranteed for display — sort explicitly when ordering matters.
- Linking a stylesheet by its source extension (e.g., `<link href="/main.scss">`) — direct source requests are blocked. Link the compiled output (`/main.css`).
