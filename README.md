[![](https://img.shields.io/github/release-date/SubtitleEdit/subtitleedit.svg)](https://github.com/nifanic/server-rendering-in-2018)
[![](https://img.shields.io/github/last-commit/google/skia.svg)](https://github.com/nifanic/server-rendering-in-2018)

# Server Rendering in 2018

> How do I do this again?

Another year, another React app supports *uninitiated server-side rendering*. In terms of ReactJS, this technique renders web app on server first and then passes it to client browser just before it starts running (rather large) ReactJS bundled code.

Server rendering is really one of those things that are great in theory when you can make it working, and terrible when things are half broken and you don’t know why.

The purpose of this post is to get you out of that “terrible” zone, so you could focus on actually building an app instead of digging deep into pre-compiled NodeJS packages all week trying to work out why things aren’t working.

## Theory: Server Rendering is Easy!

If you haven’t already got a React App, setting up a server that renders your app is fairly easy (assuming, you’re using ExpressJS.) You just use an ES6 compliant JavaScript engine to run the following:

```jsx harmony
const express = require('express');
const React = require('react');
const { renderToString } = require('react-dom/server');
const app = express();

const HelloWorld = ({ text }) => (
  <p>{text}</p>
);

app.use('/static', express.static('build'));
app.get('*', (req, res) => {
  const appHTML = renderToString(<HelloWorld text="Hello, World" />);
  res.status(200).send(`
    <html>
      <head>
        <title>My awesome server-rendered app!</title>
        <script src="/static/main.js" />
      </head>
      <body>
        <div id="app">${appHTML}</div>
      </body>
    </html>
  `)
});
```

Done, your app has got the server rendering! Or has it?

## Practice: Adding Server-Side Rendering to Existing Apps is Total Pain (as Usual)

In 2018, time has marched forward, and everything is different again. But, unsurprisingly, what hasn’t changed is that adding *server-side rendering* is still a guesswork. 

Let’s say, you’re using the best practices outlined in [`react-boilerplate`](https://github.com/react-boilerplate/react-boilerplate). But it doesn’t support *server side rendering* ([yet](https://github.com/react-boilerplate/react-boilerplate/pull/1236)), so it seems like a good place to start.

As a starter, the project has got all the latest tools: [`webpack`](https://webpack.js.org/), [`styled-components`](https://github.com/styled-components/styled-components), [`redux`](http://redux.js.org/), [`redux-saga`](https://redux-saga.github.io/redux-saga/), [`react-router`](https://github.com/ReactTraining/react-router), [`react-loadable`](https://github.com/jamiebuilds/react-loadable), to name a few. These tools are really handy, but configuration and tweaking are required to make it working with *server-side rendering*.

Off we go, I guess.

### 1. How Exactly to Run This Server Now?

You’ve probably noticed that both JSX and ES6 formats are in use. That’s great, and NodeJS supports a lot of ES6 stuff [natively](http://node.green/) by now. Except [imports](https://medium.com/the-node-js-collection/an-update-on-es6-modules-in-node-js-42c958b890c), which likely you’ve been using heavily. [`babel-node`](https://babeljs.io/docs/usage/cli/) to the rescue, right?

Not so fast: `babel-node` is really meant to be a developer-only tool as it compiles directly into memory. You [should never](https://medium.com/@Cuadraman/how-to-use-babel-for-production-5b95e7323c2f) use it for production.

Also, strict `babel-node` is not solving the problem: other ReactJS libraries you might use could depend on CSS, images,  or something else that won’t work with NodeJS natively.

Instead, we’re going to use Webpack to build a compatible server bundle, and then run it with NodeJS. So, just change entry point to your server in Webpack config, right?

```jsx harmony
{
  entry: ['server/index.js']
}
```

Not quite:

```
ERROR in ./node_modules/fsevents/node_modules/node-pre-gyp/lib/util/compile.js
Module not found: Error: Can't resolve 'child_process' in 'web/react-boilerplate-serverless/node_modules/fsevents/node_modules/node-pre-gyp/lib/util'
 @ ./node_modules/fsevents/node_modules/node-pre-gyp/lib/util/compile.js 9:9-33
 @ ./node_modules/fsevents/node_modules/node-pre-gyp/lib ^\.\/.*$
 @ ./node_modules/fsevents/node_modules/node-pre-gyp/lib/node-pre-gyp.js
 @ ./node_modules/fsevents/fsevents.js
 @ ./node_modules/chokidar/lib/fsevents-handler.js
 @ ./node_modules/chokidar/index.js
 @ ./node_modules/watchpack/lib/DirectoryWatcher.js
 @ ./node_modules/watchpack/lib/watcherManager.js
 @ ./node_modules/watchpack/lib/watchpack.js
 @ (webpack)/lib/node/NodeWatchFileSystem.js
 @ (webpack)/lib/node/NodeEnvironmentPlugin.js
 @ (webpack)/lib/webpack.js
 @ ./server/middlewares/addDevMiddlewares.js
 @ ./server/middlewares/frontendMiddleware.js
 @ ./server/server.js
 @ ./server/index.js
 @ multi ./server/index.js
```

The first problem is that your server imports ExpressJS that depends on tons of internal packages and binary modules. Webpack cannot handle these, and will throw an error.

To handle this, Webpack will ignore `node_modules` for server builds. As a result, instead of all modules being inlined, corresponding webpack bundle just wraps the existing `require` statements:

```javascript
/***/ "chalk":
/***/ (function(module, exports) {

eval("module.exports = require(\"chalk\");n");

/***/ })
```

This means a couple of things.

First, we’re [decoupling](https://github.com/smspillaz/react-boilerplate-serverless/commit/8d06ef2f238e79df4cf5f569b136479f8f823028) our *server* and *client* configurations. And we split configurations for development and production builds, too.

Second, Webpack is set to exclude all `node_modules` requirements from bundling. This can easily be done with [`webpack-node-externals`](https://github.com/smspillaz/react-boilerplate-serverless/commit/8d06ef2f238e79df4cf5f569b136479f8f823028).

Add `target: 'node'` to Webpack configuration:

```jsx harmony
entry: [
  path.join(process.cwd(), 'server/index.js'),
],

externals: [nodeExternals()],

output: {
  filename: 'server.js',
  path: path.join(process.cwd(), 'build'),
},

target: 'node',
server: true
```

#### NPM Task
To build server bundle, add these scripts to your `package.json`:

```json
"build:dev:server": "cross-env NODE_ENV=development webpack --config internals/webpack/webpack.dev.server.babel.js --color --progress",
"build:server": "cross-env NODE_ENV=production webpack --config internals/webpack/webpack.prod.server.babel.js --color -p --progress --hide-modules --display-optimization-bailout",
```

You'll notice different options for the development and production builds. (In short, use `-p` flag for production, to turn on *UglifyJS*, *optimisation*, *code splitting*, etc.)

#### Webpack DLL

Next, if you’re building a [Webpack DLL](https://webpack.js.org/plugins/dll-plugin/) (like `react-boilerplate`), [exclude](https://github.com/smspillaz/react-boilerplate-serverless/commit/721b695f2e9dc81bcbf7ef82492f25daabfcd1f6) any *server-only* dependencies from it, as the DLL is for client.

#### `DefinePlugin`

ReactJS code should be divided into *server-only* and *client-only* chunks. `DefinePlugin` is the best practice to do so.

The `__SERVER__` variable helps to separate *client-only* code from *server-only* one:

```javascript
plugins: options.plugins.concat([new webpack.DefinePlugin({
  // Put a define in place if we're server-side rendering
  ...(options.server ? {
    __SERVER__: true,
  } : {
    __SERVER__: false,
  }),
})])
```

### 2. Server-Unfriendly Things to Avoid

There are a couple of things you can’t do in your bundle for server.

#### Don’t Import JSON That is Intended for Run-time

NodeJS lets you do this  with `require()`. And Webpack handles `require()` at build-time, too. These will both be blown up as no relevant loader is configured.

It also won’t load JSON (unless generated) at run-time.

Instead, [replace](https://github.com/smspillaz/react-boilerplate-serverless/commit/3b4f40061ac7f1b970d7add7599c25be52e85b9d) `require()` with `fs.readFileSync` (or similar.)

#### Don’t Import CSS or Images Directly From Modules

Do this with [environment variables](https://github.com/KyleAMathews/react-spinkit#server-side-rendering), or [`null-loader`](https://github.com/webpack-contrib/null-loader).

#### Wrap Modules That Do Bad Things

Some modules, on `require`, may try to access browser properties.

To handle such modules:

- [wrap](https://github.com/smspillaz/intuitive-math/commit/91f96a1a0bfcc89424c596fbbd7e501c33fdf625) it with another component that defines relevant property to something sensible, and then unset it, OR

- [exclude](https://github.com/smspillaz/intuitive-math/commit/3d594f26dd98750251108ed55f121b84f79eac72) it from *Webpack DLL* with [`webpack-node-externals`](https://www.npmjs.com/package/webpack-node-externals). External server-side requires are immediately evaluated on load, giving you no opportunity to monkey patch relevant properties.

### 3. Where Have All Styles Gone?

Hopefully, you have *something* rendered on server side at this point, except you get the dreaded *flash of unstyled content* ([FOUC](https://en.wikipedia.org/wiki/Flash_of_unstyled_content)) before client-side rendering takes over.

If you're using `styled-components`, it magically injects `<style>` elements into `<head>`. (DOM should’ve been available at the time of injection.)

`styled-components` has a [helper](https://www.styled-components.com/docs/advanced#server-side-rendering) that collects all `<style>` elements, and injects it into `<head>` of server-rendered page:

```jsx harmony
const stylesheet = new ServerStyleSheet();
const html = ReactDOMServer.renderToString(
  stylesheet.collectStyles(
    <Root />
  )
);
const styleTags = stylesheet.getStyleTags();

res.status(200).send(`
  <html>
    <head>
      <title>My awesome server-rendered app!</title>
      ${styleTags}
      <script src="/static/main.js" />
    </head>
    <body>
      <div id="app">${appHTML}</div>
    </body>
  </html>
`);
```

#### Pinned Old Dependencies and `styled-components`

`styled-components` supports *server-side rendering* since version 2.0.0.

If some dependencies have been pinned to its older versions, it won’t be collected into array of `<style>` elements by `styled-components`: its older version simply doesn’t know how to insert those `<style>` elements into intermediate component created by `collectStyles`.

Thankfully, with NPM’s `resolutions` attribute, versions of installed dependencies can be specified:

```json
"resolutions": {
  "styled-components": "^2.4.0"
}
```

### 4. Server-Side Routing

If `react-router` is used, then there are slightly different configurations you need to apply in server-side case.

For client-side rendering only, you have a `browserHistory` object connected to your Redux `store`, and `ConnectedRouter` using that history. Since it depends on browser properties, it obviously won’t work on server.

For server-side rendering,
1. create `memoryHistory` object from the current request URL, and
2. inject both Redux `store` and `history` objects into your `<Root>` component. Use these instead of `browserHistory` on client side.

```jsx harmony
import createMemoryHistory from 'history/createMemoryHistory';
import { routerMiddleware } from 'react-router-redux';
import { createStore, applyMiddleware, compose } from 'redux';

. . .

configureStore = (initialState = {}, history) => {
  // 2. routerMiddleware: Syncs the location/URL path to the state
  const middlewares = [
    routerMiddleware(history),
  ];

  const enhancers = [
    applyMiddleware(...middlewares),
  ];

  const store = createStore(
    createReducer(),
    initialState,
    compose(...enhancers)
  );

  return store;
}

. . .

const memoryHistory = createMemoryHistory(req.url);
memoryHistory.push(req.originalUrl);

const store = configureStore({}, memoryHistory);

const html = ReactDOMServer.renderToString(
  <Root history={memoryHistory} store={store} />
);
```

### 5. “None of my Loadables are Visible”

Using `react-loadable` on client side is a great way to speed up page loading: it’s asynchronously loading expensive parts of your app once the “shell” is loaded.

On server side, it’s not useful since all that happens is that a pointless asycnhronous request gets fired on the server side which and by the time it resolves you've already rendered and it is too late.

Instead, what you can do is to collect up the loadables into separate `script` elements and ship them to the client on the server-side render. Unfortunately, this is not quite as simple as it looks—there’s a few things that need to be done here in order to make this work.

#### Babel Plugin

Add [`react-loadable/babel`](https://github.com/smspillaz/react-boilerplate-serverless/commit/b34e303d0ddba620eb478ec44acc37e3a13f3400) to your `.babelrc`:

```json
"plugins": [
  "react-loadable/babel"
]
```

#### Webpack Plugin

Use [`ReactLoadablePlugin`](https://github.com/smspillaz/react-boilerplate-serverless/commit/045fa12305f31c9c55f2d04c1d09164b0089c0c4) in your client-side Webpack build config to generate a manifest of Webpack chunks corresponding to each module. We'll read this file on the server side to inject `script` tags for your loadables.

`ModulesConcatenationPlugin`

As of March 1st, 2018, `react-loadable` hasn’t been fixed the compatibility issue with this plugin, so it may need to be [disabled](https://github.com/smspillaz/react-boilerplate-serverless/commit/257ebd36865036cc4e7c14c4824d9ab2d6824bbe).

#### Split Out Manifest Bootstrap

Since we'll be preloading the chunks before your main manifest, we'll need to preload the Webpack manifest bootstrap code before preloading those chunks! This can easily be done with `CommonChunksPlugin` in your client Webpack config (note that.)

```javascript
new webpack.optimize.CommonsChunkPlugin({
  name: 'manifest',
  filename: 'manifest.js',
  minChunks: Infinity,
})
```

#### `HTMLWebpackPlugin`

If `HTMLWebpackPlugin` is used to build HTML file, prevent  `HtmlWebpackPlugin` from including the manifest chunk, since we manually include it in the right place later.

```javascript
new HtmlWebpackPlugin({
  inject: true, // Inject all files that are generated by webpack, e.g. bundle.js
  template: 'app/index.html',
  excludeChunks: ['manifest'],
})
```

#### Server-Side Renderer

Now you can read the manifest into your server-side renderer and capture the loadables that would have been requested on the given route and preload them into `<script>` tags on the rendered page.

```jsx harmony
const modules = [];
const html = ReactDOMServer.renderToString(
  <Loadable.Capture report={(moduleName) => modules.push(moduleName)}>
    <Root />
  </Loadable.Capture>
);

fs.readFile('./build/react-loadable.json', 'utf-8', (statsErr, statsData) => {
  const bundles = getBundles(JSON.parse(statsData), modules);
  const bundlesHTML = bundles.map((bundle) =>
    `<script src="/static/${bundle.file}"></script>`
  ).join('\n');

  res.status(200).send(`
    <html>
      <head>
        <title>My awesome server-rendered app!</title>
        <script src="/static/manifest.js">
        ${bundlesHTML}
        <script src="/static/main.js" />
      </head>
      <body>
        <div id="app">${appHTML}</div>
      </body>
    </html>
  `);
});
```

#### Client Side `preloadReady`

Prevent client side from  any renderings until all bundles are pre-loaded:

```javascript
Loadable.preloadReady().then(() => {
  ReactDOM.render(
    <Root />,
    MOUNT_NODE
  );
});
```

### 6. Running Your `redux-sagas` Before Rendering

Usually, when components are mounted, there's a bunch of data you might want to immediately start fetching from the server. *If* it is cheap to do so, it might be beneficial for you to do some of that work on the server to avoid a round trip. To do that, you'll want to wait for your Redux sagas to complete until final result is rendered.

The gist of what will happen here is that we'll kick off a render of your application, run any sagas which need to be run, dispatch a special `END` event, which causes the saga generators to terminate, then render your application again, this time with an updated redux store containing pre-filled data.

```jsx harmony
import { END } from 'redux-saga';

import { createStore, applyMiddleware, compose } from 'redux';
import createSagaMiddleware from 'redux-saga';

const sagaMiddleware = createSagaMiddleware();

export default function configureStore(initialState = {}) {
  const middlewares = [
    sagaMiddleware
  ];

  const enhancers = [
    applyMiddleware(...middlewares),
  ];

  const store = createStore(
    initialState,
    compose(...enhancers)
  );

  // Extensions
  store.runSaga = sagaMiddleware.run;
  store.injectedSagas = {}; // Saga registry

  return store;
}

const store = configureStore({});
ReactDOM.renderToString(<Root store={store} />);
store.dispatch(END);

store.runSagas(sagas).then(() => {
  const html = ReactDOM.renderToString(<Root store={store}>)
});
```

### 7. Redux State Hydration

As of now, content on the page has been rendered, except the server side one.

You’ve set some Redux state the client knows nothing about! That means that when the client side re-renders it'll do so without the benefit of that state and probably means that React won't be able to re-use a lot of your server-rendered markup.

What you'll need to do is *serialize* the Redux state and send that over to the client such that the client can "rehydrate" from that state and continue where the server left off.

Thankfully, that's pretty straightforward. Just encode the server state as JSON and assign it to variable in a `<script>` tag that gets executed on the server side:

```jsx harmony
const stateHydrationHTML = (
  `<script>window.__SERVER_STATE = ${JSON.stringify(store.getState())}</script>`
);

res.status(200).send(`
  <html>
    <head>
      <title>My awesome server-rendered app!</title>
      ${stateHydrationHTML}
      <script src="/static/main.js" />
    </head>
    <body>
      <div id="app">${appHTML}</div>
    </body>
  </html>
`);
```

#### Client Side

1. Read the `__SERVER_STATE` property (if exists), and
2. Initialize the store from there:

```jsx harmony
const initialState = window.__SERVER_STATE || {};
const store = configureStore(initialState);
```

### 8. Serverless

If you’re building a [server-side serverless](https://smspillaz.wordpress.com/2017/11/19/serverless-react-boilerplate/)   ReactJS app, you need to create a Webpack bundle for the serverless deployment, too.

Thankfully, that’s just a matter of making a “Webpack library” out of your existing serverless entry point:

```javascript
{
  entry: [
    path.join(process.cwd(), 'lambda.js'),
  ],

  externals: [nodeExternals()],

  output: {
    filename: 'prodLambda.js',
    libraryTarget: 'umd',
    library: 'lambda',
    // Needs to go in process.cwd in order to be imported
    // correctly from lambda
    path: path.join(process.cwd())
  },

  target: 'node'
}
```

## Demo

To shorten your learning curve, I’ve prepped [`react-boilerplate-serverless`](https://github.com/smspillaz/react-boilerplate-serverless/commits/server-rendering), a working implementation of *server-side rendering* feature.

Feel free to check it out and fork.

*Thanks to [Jack Scott](https://github.com/jackrobertscott) for reviewing a draft of this post.*