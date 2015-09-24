# `offline-plugin` for webpack

This plugin is intended to provide offline experience for **webpack** projects. It uses **ServiceWorker** and **AppCache** as a fallback under the hood. Simply include this plugin in your ``webpack.config`` and your project will became offline ready by caching all (or some) output assets.

## Install

`npm install offline-plugin [--save-dev]`

## Setup

```js
// webpack.config.js example

var OfflinePlugin = require('offline-plugin');

module.exports = {
  // ...

  plugins: [
    // ... other plugins

    new OfflinePlugin({
      // All options are optional
      caches: 'all',
      scope: '/',
      updateStrategy: 'all',
      version: 'v1',

      ServiceWorker: {
        output: 'sw.js'
      },

      AppCache: {
        directory: 'appcache/'
      }
    })
  ]
  // ...
}

```

## Options

All options are optional and `offline-plugin` could be used without specifying them. Also see full list of default options [here](https://github.com/NekR/offline-plugin/blob/master/src/index.js#L9).

* **caches**: `'all' | Object`. Tells to the plugin what to cache and how. Default: `'all'`.
  * `all`: means that everything (all the webpack output assets) will be cached in `main` cache.
  * `Object`: Object with 3 possible _Array<string>_ sections (properties). All sections are optional and default to `[':rest:']`. `:rest:` is a special keyword which means "_all the rest unused (uncahed) assets_". To match multiple assets or assets with dynamic names, use [pattern matching](https://www.npmjs.com/package/minimatch). **Any section can contain any assets, not only those which are generated by the _webpack_ build.** However, "_pattern matching_" is executed only against generated assets. When external asset is included you will receive warning about it (missing from generated assets) unless it's explicitly marked as _external_ via `externals` option.
    * `main` cache. Assets listed in this section are cached first (via `install` event in `ServiceWorker`) and if caching of this section fails -- nor assets are cached at all. Hence, it should contain minimal set of assets, for example `['index.html', 'main.js']`.
    * `additional` cache. Assets in this section are loaded after `main` section is successfully loaded. In `ServiceWorker` this section is loaded in `activate` event. If any of assets fails to download, then nothing is cached from the `additional` section and all the assets are moved to the `optional` section. If **current update strategy** is `changed`, then only failed to download assets are moved to the `optional` section, all other are successfully cached.
    * `optional` cache. This sections **is only supported** by `ServiceWorker`. As you may guess from its name, assets are cached only when they are fetched from the server. `ServiceWorker` won't download them ahead of time.
* **scope**: `string`. Scope of the project, for example `'/m/'` or `'/admin/'`. Default: `'/'`.
* **updateStrategy**: `'all' | 'hash' | 'changed'`. Cache update strategy. Default: `'all'`.
  * `all` strategy uses `version` passed to options as a cache tag. When version changes, old version cache is removed and new files are downloaded. Of course if files has same name were not changed (304 status returned), then probably browser won't download them again and will just update cache.
  * `hash` strategy is basically the same as `all` strategy, but uses _webpack unique compilation hash_ as a tag.
  * `changed` strategy is more advanced than `all` or `hash`. To work properly it requires output assets to have unique names between compilations, e.g. _file's unique hash_ in its name. **With this strategy enabled, `index.html` file (or other files without dynamic name) should be placed in `main` section of the cache, otherwise they won't be revalidated**.
    * For `ServiceWorker` this means that only new files will be downloaded and missing files deleted from the cache.
    * For `AppCache` it's basically same as previous strategies since `AppCache` revalidates all the assets. 304 HTTP status rule of course still works.
* **externals**: `Array<string>`. Explicitly marks cache asset as _external_ so you won't receive any warnings about it if asset it missing from _webpack generated assets_.
* **version**: `string | Function`. Version of the cache. Is used only with `all` update strategy. Might be a function, useful in _watch-mode_ when you need dynamic date for each generation. Default: _Current date_.
* **rewrites**: `Function | Object`. Rewrite function or rewrite map (`Object`). Useful when assets are server in a different way from the client perspective, e.g. usually `index.html` served as `/`. Default: _function which rewrites_ `/any/path/index.html` _to_ `/any/path/`.
* **ServiceWorker**: `Object | null | false`. Settings for the `ServiceWorker` cache. Use `null` or `false` to disable `ServiceWorker` generation.
  * `output`: `string`. Relative (from the _webpack_'s config `output.path`) output path for emitted script. Default: `'sw.js'`.
  * `entry`: `string`. Relative or absolute path to file which will be used as `ServiceWorker` entry. Useful to implement additional function for it. Default: _empty file_.
* **AppCache**: `Object | null | false`. Settings for the `AppCache` cache. Use `null` or `false` to disable `AppCache` generation.
  * `NETWORK`: `string`. Reflects `AppCache`'s `NETWORK` section. Default: `'*'`.
  * `directory`: `string`. Relative (from the _webpack_'s config `output.path`) output directly path for the `AppCache` files. Default: `'appcache/'`.
  * `caches`: `Array<string>`. List of sections from `caches` option (`main`, `additional` or `optional`). Default: `['main', 'additional']`.

## Runtime

Besides plugin configuration, you also need to initialize it at runtime phase. This is most easier part:

```js
require('offline-plugin/runtime').install();
```

Runtime `install` accepts 3 optional arguments: `options`, `successCallback`, `errorCallback`. `options` is not used at the moment (ignored), so it's better to just put `null` instead.

## FAQ

**Is it possible to minify `ServiceWorker` script output?**  
Yes, `offline-plugin` perfectly works with official `webpack.optimize.UglifyJsPlugin`, so if it's used your will get minified `ServiceWorker` script as well (no additional options required).

**Is there a way to match assets with dynamic file names, like compilation hash or version?**  
Yes, it's possible with `pattern matching`, which is performed by [minimatch](https://www.npmjs.com/package/minimatch) library.  
Example: ``main: ['index.html', 'scripts/main.*.js']``.

**Do I need to include [cache-polyfill](https://github.com/coonsta/cache-polyfill) for the `ServiceWorker`?**  
No, it's included automatically for you.


## License

[MIT](LICENSE.md)


## CHANGELOG

### 1.2

Remove support of multi-stage caching from `AppCache`. Reason is that files cached in second manifest cannot be accessed from page cached by first one, since `NETWORK` section can only dictate to use _network_ (`*`) or _nothing_ (pretend offline), but not _fallback to browser defaults_. This means that any attempt to access files of second manifest goes to the network or fails immediately, instead of reading from cache.

### 1.1

Fix `ServiceWorker` login to not cache `additional`'s section assets on `activate` event, instead, cache them without blocking any events. Other `ServiceWorker` logic fixes.

### 1.0

Release