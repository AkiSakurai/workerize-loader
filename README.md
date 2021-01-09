<img src="https://i.imgur.com/HZZG8wr.jpg" width="1358" alt="workerize-loader">

# workerize-loader [![npm](https://img.shields.io/npm/v/workerize-loader.svg?style=flat)](https://www.npmjs.org/package/workerize-loader) [![travis](https://travis-ci.org/developit/workerize-loader.svg?branch=master)](https://travis-ci.org/developit/workerize-loader)

> A webpack loader that moves a module and its dependencies into a Web Worker, automatically reflecting exported functions as asynchronous proxies.

- Bundles a tiny, purpose-built RPC implementation into your app
- If exported module methods are already async, signature is unchanged
- Supports synchronous and asynchronous worker functions
- Works beautifully with async/await
- Imported value is instantiable, just a decorated `Worker`


## Install

```sh
npm install -D workerize-loader
```


### Usage

**worker.js**:

```js
// block for `time` ms, then return the number of loops we could run in that time:
export function expensive(time) {
    let start = Date.now(),
        count = 0
    while (Date.now() - start < time) count++
    return count
}
```

**index.js**: _(our demo)_

```js
import worker from 'workerize-loader!./worker'

let instance = worker()  // `new` is optional

instance.expensive(1000).then( count => {
    console.log(`Ran ${count} loops`)
})
```

### Options

Workerize options can either be defined in your Webpack configuration, or using Webpack's [syntax for inline loader options](https://webpack.js.org/concepts/loaders/#inline).

#### `inline`

Type: `Boolean`
Default: `false`

You can also inline the worker as a BLOB with the `inline` parameter

```js
// webpack.config.js
{
  loader: 'workerize-loader',
  options: { inline: true }
}
```

or

```js
import worker from 'workerize-loader?inline!./worker'
```

#### `name`

Type: `String`
Default: `[hash]`

Customize filename generation for worker bundles. Note that a `.worker` suffix will be injected automatically (`{name}.worker.js`).

```js
// webpack.config.js
{
  loader: 'workerize-loader',
  options: { name: '[name].[contenthash:8]' }
}
```

or

```js
import worker from 'workerize-loader?name=[name].[contenthash:8]!./worker'
```

#### `publicPath`

Type: `String`
Default: based on `output.publicPath`

Workerize uses the configured value of `output.publicPath` from Webpack unless specified here. The value of `publicPath` gets prepended to bundle filenames to get their full URL. It can be a path, or a full URL with host.

```js
// webpack.config.js
{
  loader: 'workerize-loader',
  options: { publicPath: '/static/' }
}
```

#### `ready`

Type: `Boolean`
Default: `false`

If `true`, the imported "workerized" module will include a `ready` property, which is a Promise that resolves once the Worker has been loaded. Note: this is unnecessary in most cases, since worker methods can be called prior to the worker being loaded.

```js
// webpack.config.js
{
  loader: 'workerize-loader',
  options: { ready: true }
}
```

or

```js
import worker from 'workerize-loader?ready!./worker'

let instance = worker()  // `new` is optional
await instance.ready
```

#### `import`

Type: `Boolean`
Default: `false`

When enabled, generated output will create your Workers using a Data URL that loads your code via `importScripts` (eg: `new Worker('data:,importScripts("url")')`). This workaround enables cross-origin script preloading, but Workers are created on an "opaque origin" and cannot access resources on the origin of their host page without CORS enabled. Only enable it if you understand this and specifically need the workaround.

```js
// webpack.config.js
{
  loader: 'workerize-loader',
  options: { import: true }
}
```

or 

```js
import worker from 'workerize-loader?import!./worker'
```

### About [Babel](https://babeljs.io/)

If you're using [Babel](https://babeljs.io/) in your build, make sure you disabled commonJS transform. Otherwize, workerize-loader won't be able to retrieve the list of exported function from your worker script :
```js
{
    test: /\.js$/,
    loader: "babel-loader",
    options: {
        presets: [
            [
                "env",
                {
                    modules: false,
                },
            ],
        ]
    }
}
```

### Polyfill Required for IE11

Workerize-loader supports browsers that support Web Workers - that's IE10+.
However, these browsers require a polyfill in order to use Promises, which Workerize-loader relies on.
It is recommended that the polyfill be installed globally, since Webpack itself also needs Promises to load bundles.

The smallest implementation is the one we recommend installing:

`npm i promise-polyfill`

Then, in the module you are "workerizing", just add it as your first import:

```js
import 'promise-polyfill/src/polyfill';
```

All worker code can now use Promises. 

### Testing

To test a module that is normally imported via `workerize-loader` when not using Webpack, import the module directly in your test:

```diff
-const worker = require('workerize-loader!./worker.js');
+const worker = () => require('./worker.js');

const instance = worker();
```

To test modules that rely on workerized imports when not using Webpack, you'll need to dig into your test runner a bit. For Jest, it's possible to define a custom `transform` that emulates workerize-loader on the main thread:

```js
// in your Jest configuration
{
  "transform": {
    "workerize-loader(\\?.*)?!(.*)": "<rootDir>/workerize-jest.js"
  }
}
```

... then add the `workerize-jest.js` shim to your project:

```js
module.exports = {
  process(src, filename, config, options) {
    return 'module.exports = () => require(' + JSON.stringify(filename.replace(/.+!/,'')) + ')';
  },
};
```

Now your tests and any modules they import can use `workerize-loader!` prefixes.

### Credit

The inner workings here are heavily inspired by [worker-loader](https://github.com/webpack-contrib/worker-loader). It's worth a read!


### License

[MIT License](https://oss.ninja/mit/developit) © [Jason Miller](https://jasonformat.com)
