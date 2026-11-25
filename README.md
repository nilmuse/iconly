# iconly

`iconly` gives you two ways to use SVG icon sprites in the browser: load and cache an external sprite file, or build one from your own icon objects without giving up tree-shaking.

[![npm](https://img.shields.io/npm/v/iconly.svg?colorB=brightgreen)](https://www.npmjs.com/package/iconly)
[![NPM Downloads](https://img.shields.io/npm/dm/iconly.svg?style=flat)](https://www.npmjs.com/package/iconly)

[Demo](https://codepen.io/ux-ui/pen/zYmyqWR)

## Installation

```bash
npm install iconly
```

## Quick start

Both examples below use a browser app with a bundler. Run the entry module after the page markup exists.

### External sprite file

Save this sprite as `public/sprite.svg`, where your app serves public files from the site root:

```svg
<svg xmlns="http://www.w3.org/2000/svg">
  <symbol id="search" viewBox="0 0 24 24">
    <circle cx="10" cy="10" r="6" fill="none" stroke="currentColor" stroke-width="2" />
    <path d="m15 15 6 6" fill="none" stroke="currentColor" stroke-width="2" />
  </symbol>
</svg>
```

Add an icon to your page:

```html
<svg width="24" height="24" aria-label="Search" role="img">
  <use href="#search"></use>
</svg>
```

In your app entry:

```ts
import { createIconly } from 'iconly';

const iconLoader = createIconly({
  file: '/sprite.svg',
  version: '1.0',
});

const result = await iconLoader.init();

if (!result.ok) {
  console.error(result.error);
}
```

When `init()` succeeds, the `<use>` element can reference the inserted symbol. iconly wraps the sprite in an off-screen `<div data-iconly="iconset" aria-hidden="true">`. The default container is `document.body`, with `document.documentElement` as a fallback if the body is not available.

### Icon objects

To build a sprite from JavaScript objects, use `iconly/sprite` instead of the loader. This path does not fetch files or use storage. With the same `<svg>` markup above, replace the entry code with:

```ts
import { createSprite } from 'iconly/sprite';

const search = {
  name: 'search',
  viewBox: '0 0 24 24',
  body: `
    <circle cx="10" cy="10" r="6" fill="none" stroke="currentColor" stroke-width="2" />
    <path d="m15 15 6 6" fill="none" stroke="currentColor" stroke-width="2" />
  `,
};

const sprite = createSprite({
  icons: [search],
});

const result = sprite.render();

if (!result.ok) {
  console.error(result.error);
}
```

Each icon has a `name` used as its symbol ID, a `viewBox`, and an SVG `body`. Keep icons in separate ESM exports if you want your bundler to remove unused icons. iconly assembles only the objects you pass to it.

## API

### Methods

From `iconly`:

| Method | Result | Behavior |
| --- | --- | --- |
| `createIconly(config?)` | `IconlyInstance` | Creates a loader. |
| `iconLoader.init()` | `Promise<Result<void>>` | Loads the sprite from cache or the network and inserts it into the DOM. |
| `iconLoader.abort()` | `void` | Cancels an in-flight fetch. |

From `iconly/sprite`:

| Method | Result | Behavior |
| --- | --- | --- |
| `createSprite(config)` | `SpriteInstance` | Creates a builder for the supplied icon objects. |
| `sprite.render()` | `Result<void>` | Builds and inserts the sprite synchronously. |
| `buildSpriteString(icons)` | `string` | Builds raw SVG without accessing the DOM. |

Both DOM insertion methods reuse the first `[data-iconly="iconset"]` wrapper in the configured container, replacing its contents. Use separate containers if you need to keep multiple sprites on the page.

### Options

#### createIconly

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `file` | `string` | `'./icons.svg'` | URL of the SVG file containing the icons. |
| `version` | `string` | `'1.0'` | Version of the icon set. |
| `debug` | `boolean` | `false` | Enables debug callbacks and logger debug output. |
| `container` | `string \| HTMLElement` | `document.body` or `document.documentElement` | Container where the sprite is inserted. |
| `storage` | `'indexeddb' \| 'memory' \| 'session' \| IconStorage` | `'indexeddb'` | Storage strategy or custom storage implementation. |
| `dbName` | `string` | `'iconlyDB'` | IndexedDB database name (for `indexeddb`). |
| `storeName` | `string` | `'icons'` | IndexedDB store name (for `indexeddb`). |
| `sessionKeyPrefix` | `string` | `'iconly'` | SessionStorage key prefix (for `session`). |
| `sanitize` | `(svg: string) => string` | `undefined` | Optional hook to sanitize SVG before parsing. Runs before built-in hardening. |
| `logger` | `Logger` | `undefined` | Optional logger with `debug` and `error` methods. |
| `onError` | `(error: IconlyError) => void` | `undefined` | Receives fatal and recoverable errors. |
| `onDebug` | `(...messages: unknown[]) => void` | `undefined` | Receives debug messages when `debug` is `true`. |

Both logger methods are optional and accept `...messages: unknown[]`. Errors reach `logger.error` even when `debug` is `false`.

#### createSprite

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `icons` | `IconlyIcon[]` | Required | Icon objects to include in the sprite. |
| `container` | `string \| HTMLElement` | `document.body` or `document.documentElement` | Container where the sprite is inserted. |
| `sanitize` | `(svg: string) => string` | `undefined` | Optional hook to sanitize SVG before parsing. Runs before built-in hardening. |

### Results and errors

`init()` and `render()` return failures as `Result` values rather than throwing:

```ts
type Result<T> =
  | { ok: true; value: T }
  | { ok: false; error: IconlyError };

interface IconlyError {
  code: IconlyErrorCode;
  message: string;
  cause?: unknown;
}
```

Fatal and recoverable cache errors trigger `onError` and `logger.error` when provided, even when debugging is off. A cache error can therefore be reported while `init()` returns `ok: true`.

Calling `iconLoader.abort()` cancels an in-flight fetch with the error code `fetch_aborted`.

## Build SVG without the DOM

For SSR or tests, `buildSpriteString()` returns raw SVG without rendering it. Using the `search` object from Quick start:

```ts
import { buildSpriteString } from 'iconly/sprite';

const svg = buildSpriteString([search]);
```

The output is not sanitized. Sanitize it before inserting it into HTML unless you trust the source icons; see [SVG content](#svg-content).

## Cache behavior

- Each cache entry is keyed by the normalized `file` URL and `version`, so different sprites can safely share storage on the same origin.
- Caching is best-effort. Storage read failures do not prevent a network fetch, and write failures do not undo an inserted sprite. Errors are reported through `onError` and `logger.error`.
- If cached data is not a valid SVG document, iconly replaces it with a fresh copy from the network.
- A fetched sprite is cached only after it has been parsed, hardened, and inserted successfully.
- If you implement a custom `IconStorage`, treat both the key passed to `get()` and the value returned in `IconRecord.version` as opaque.

## SVG content

iconly inserts SVG into the live DOM. Use sprite files and icon objects you trust, or sanitize them before insertion.

- Do not use `file` URLs or `icon.body` values from untrusted user input without sanitizing them first.
- Before insertion, `createIconly` and `createSprite` strip event-handler attributes (`on*`), `<script>`, `<foreignObject>`, `javascript:` / `data:text/html` links, and SMIL animation elements targeting `href`.
- The built-in hardening is not a full HTML sanitizer. It does not cover every vector, including CSS in `<style>` or external references in `<use>` / `<image>`. For untrusted SVG, provide a `sanitize` hook backed by a dedicated library such as [DOMPurify](https://github.com/cure53/DOMPurify).

After installing DOMPurify in your app, this replaces the loader configuration from Quick start. The file URL is still an application-controlled URL:

```ts
import DOMPurify from 'dompurify';
import { createIconly } from 'iconly';

const iconLoader = createIconly({
  file: '/sprite.svg',
  sanitize: (svg) =>
    DOMPurify.sanitize(svg, { USE_PROFILES: { svg: true, svgFilters: true } }),
});
```

In `iconly/sprite`, `name` and `viewBox` are escaped in attributes. The `body` value is inserted as-is and is sanitized only by `createSprite().render()` or by your own sanitizer.

## Browser support

- Chrome and Edge 90+
- Firefox 90+
- Safari and iOS Safari 15+
- Current Chrome for Android

Internet Explorer and Legacy Edge are not supported. The external-sprite loader relies on native `fetch`, `AbortController`, `DOMParser`, and the selected storage API. If IndexedDB or SessionStorage is unavailable, the sprite still loads, but without caching.

## License

MIT
