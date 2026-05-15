---
name: nodebb-v4-plugin
description: Use when developing NodeBB v4.x plugins — covers plugin structure, hook system, ACP settings pages, templates, i18n, admin routes, testing, and publishing. Use whenever the user mentions NodeBB plugin, NodeBB hook, NodeBB admin page, NodeBB upload provider, or wants to extend NodeBB functionality.
---

# NodeBB v4.x Plugin Development

## Quick Start

Every NodeBB plugin needs at minimum:
```
my-plugin/
├── package.json
├── plugin.json      # hooks, settings, metadata
├── library.js       # main logic, exports hook methods
└── templates/       # Benchpress .tpl files (optional)
```

## plugin.json

```json
{
  "id": "nodebb-plugin-my-plugin",
  "name": "My Plugin",
  "description": "What it does",
  "url": "https://github.com/user/repo",
  "library": "library.js",
  "settingsRoute": "/admin/plugins/my-plugin",
  "acpScripts": ["static/lib/admin.js"],
  "languages": "language",
  "defaultLang": "en-GB",
  "templates": "templates",
  "hooks": [
    { "hook": "static:app.load", "method": "init" },
    { "hook": "filter:admin.header.build", "method": "addAdminLink" }
  ],
  "nbbpm": { "compatibility": "^4.0.0" }
}
```

Key fields:
- `settingsRoute` — adds a "Settings" button on the plugins list page
- `acpScripts` — client-side JS bundled into `scripts-admin.js`
- `languages` — directory containing i18n JSON files (relative to plugin root)
- `defaultLang` — fallback language code (default: `en-GB`)
- `templates` — directory containing Benchpress templates (default: `templates`)

## Hook System

Hook types:
- `filter:` — receives params, must return (possibly modified) params. Sequential chain.
- `action:` — fire-and-forget, no return value.
- `static:` — startup events (e.g., `static:app.load` for route registration).

Priority: lower number runs first (default: 10).

## library.js Pattern

```js
'use strict';

const winston = require.main.require('winston');
const meta = require.main.require('./src/meta');
const helpers = require.main.require('./src/routes/helpers'); // NOT controllers/helpers

const plugin = {};
const SETTINGS_HASH = 'nodebb-plugin-my-plugin';

// Admin route registration
plugin.init = async (params) => {
  const { router, middleware } = params;
  helpers.setupAdminPageRoute(router, '/admin/plugins/my-plugin', middleware, [], async (req, res) => {
    const settings = await meta.settings.get(SETTINGS_HASH);
    res.render('admin/plugins/my-plugin', { settings });
  });
};

// Sidebar navigation
plugin.addAdminLink = async (custom_header) => {
  custom_header.plugins.push({
    name: 'My Plugin',
    route: '/plugins/my-plugin',  // NOT /admin/plugins/... (sidebar template prepends /admin)
  });
  return custom_header;
};

module.exports = plugin;
```

### CRITICAL: helpers import path

Use `require.main.require('./src/routes/helpers')` for `setupAdminPageRoute`.
`./src/controllers/helpers` does NOT have this function — it will silently fail.

### CRITICAL: sidebar nav route

The sidebar template renders `{relative_path}/admin{./route}`. Your route must NOT include `/admin` prefix. Use `/plugins/my-plugin`, not `/admin/plugins/my-plugin`.

## ACP Settings Page

### library.js — register route and render template

```js
plugin.init = async (params) => {
  const { router, middleware } = params;
  helpers.setupAdminPageRoute(router, '/admin/plugins/my-plugin', middleware, [], async (req, res) => {
    const settings = await meta.settings.get(SETTINGS_HASH);
    res.render('admin/plugins/my-plugin', { settings });
  });
};
```

### Template — use `data-key` for plugin settings

Templates use Benchpress syntax. Place at `templates/admin/plugins/my-plugin.tpl` (NOT `templates/admin/plugins/my-plugin/settings.tpl`).

```html
<div class="acp-page-container">
  <!-- IMPORT admin/partials/settings/header.tpl -->
  <div class="row settings m-0">
    <div id="spy-container" class="col-12 col-md-8 px-0 mb-4" tabindex="0">
      <div class="mb-3">
        <label class="form-label" for="apiKey">[[my-plugin:api-key]]</label>
        <input type="text" class="form-control" id="apiKey" data-key="apiKey" />
      </div>
      <div class="form-check form-switch mb-3">
        <input class="form-check-input" type="checkbox" id="enabled" data-key="enabled" />
        <label class="form-check-label" for="enabled">[[my-plugin:enabled]]</label>
      </div>
    </div>
    <!-- IMPORT admin/partials/settings/toc.tpl -->
  </div>
</div>
```

### Client-side JS — Settings.sync

Place at `static/lib/admin.js`. Reference via `acpScripts` in plugin.json.

```js
'use strict';

define('admin/plugins/my-plugin', ['settings'], function (Settings) {
  const ACP = {};
  ACP.init = function () {
    Settings.sync('nodebb-plugin-my-plugin', $('#spy-container'));
    $('#save').on('click', function () {
      Settings.save('nodebb-plugin-my-plugin', $('#spy-container'), function () {
        app.alert({ type: 'success', alert_id: 'my-plugin-saved', title: 'Settings Saved', timeout: 3000 });
      });
    });
  };
  return ACP;
});
```

### Reading settings in library.js

```js
const settings = await meta.settings.get('nodebb-plugin-my-plugin');
// settings.apiKey, settings.enabled, etc.
// Checkbox values come as 'on'/true/false — use a parseBool helper
```

## Template Engine (Benchpress)

Syntax:
- `{variable}` — interpolation
- `{{{ if condition }}}...{{{ end }}}` — conditional
- `{{{ each items }}}...{{{ end }}}` — loop
- `<!-- IMPORT path/to/partial.tpl -->` — partial inclusion
- `[[namespace:key]]` — i18n string reference
- `[[namespace:key, param1, param2]]` — i18n with parameters (%1, %2)

### Template path resolution

NodeBB looks for templates at `node_modules/<plugin-id>/<templates field or 'templates'>`.

`res.render('admin/plugins/my-plugin', data)` looks for `admin/plugins/my-plugin.tpl` in that directory.

NOT `admin/plugins/my-plugin/index.tpl` or `admin/plugins/my-plugin/settings.tpl`.

## i18n

### Directory structure

```
language/
  en-GB/
    nodebb-plugin-my-plugin.json
  zh-CN/
    nodebb-plugin-my-plugin.json
```

### JSON format

Flat key-value pairs. Use `%1`, `%2` for positional parameters.

```json
{
  "settings-title": "My Plugin Settings",
  "greeting": "Hello %1, welcome to %2!"
}
```

### In templates

```
[[nodebb-plugin-my-plugin:settings-title]]
[[nodebb-plugin-my-plugin:greeting, {username}, {sitename}]]
```

Register in plugin.json: `"languages": "language"`, `"defaultLang": "en-GB"`.

## Upload Hooks

To intercept file uploads:

```json
{ "hook": "filter:uploadImage", "method": "uploadImage", "priority": 1 },
{ "hook": "filter:uploadFile", "method": "uploadFile", "priority": 1 }
```

Hook receives `{ image/file, uid, folder }`. Must return `{ url, name }`.
- `image.path` is a temp file (deleted after hook returns)
- Return absolute URLs starting with `https://` to avoid `relative_path` prepending
- Use `fileModule.allowedExtensions()` for extension validation
- Use `meta.config.maximumFileSize` for size validation

## Cleanup Hooks

For cleaning up stored objects when content is deleted:

```json
{ "hook": "action:posts.purge", "method": "onPostsPurge" },
{ "hook": "action:post.delete", "method": "onPostDelete" },
{ "hook": "filter:post.edit", "method": "onPostEditFilter" },
{ "hook": "action:post.edit", "method": "onPostEditAction" },
{ "hook": "action:topics.purge", "method": "onTopicPurge" }
```

Pattern for edit cleanup:
1. `filter:post.edit` (before) — snapshot current state to DB with TTL
2. `action:post.edit` (after) — compare before/after, delete orphaned objects

For multi-instance safety, use shared DB (with TTL) instead of in-memory maps.

## Settings Change Listener

Invalidate cached settings/S3 clients when settings change:

```json
{ "hook": "action:settings.set", "method": "onSettingsSet" }
```

```js
plugin.onSettingsSet = async () => {
  plugin._settings = null;  // clear cache
};
```

## Testing

Use Mocha + assert (NodeBB standard).

- Unit tests: test pure utility functions directly (no NodeBB dependency)
- Integration tests: use real services with environment variables

```json
"scripts": {
  "test": "mocha test/**/*.test.js --timeout 30000",
  "test:unit": "mocha test/settings.test.js",
  "test:integration": "mocha test/upload.test.js --timeout 60000"
}
```

Extract pure functions to `lib/utils.js` for testability (library.js requires NodeBB modules that aren't available in standalone tests).

## Publishing

### package.json

```json
{
  "name": "nodebb-plugin-my-plugin",
  "main": "library.js",
  "files": ["library.js", "lib/", "static/", "templates/", "language/", "plugin.json"],
  "nbbpm": { "compatibility": "^4.0.0" }
}
```

### GitHub Actions (auto-publish on tag)

```yaml
# .github/workflows/npm-publish.yml
name: Publish to npm
on:
  push:
    tags: ['v*']
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm test
      - run: npm publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
```

### Release workflow

```bash
npm version patch   # or minor/major
git push --follow-tags
```

### npm token setup

1. Create Granular Access Token at npmjs.com (with "bypass 2FA" if 2FA enabled)
2. `gh secret set NPM_TOKEN -b 'token'`

## Common Pitfalls

1. **`helpers` import** — Use `./src/routes/helpers`, not `./src/controllers/helpers`
2. **Template path** — `templates/admin/plugins/my-plugin.tpl`, not `templates/admin/plugins/my-plugin/settings.tpl`
3. **Sidebar route** — Use `/plugins/my-plugin`, not `/admin/plugins/my-plugin`
4. **Checkbox values** — Come as `'on'` (string) or boolean, need `parseBool` helper
5. **Template name** — `res.render('admin/plugins/my-plugin')` looks for `admin/plugins/my-plugin.tpl`, not `admin/plugins/my-plugin/index.tpl`
