# Reduce Your Bundle Sizes

If your bundle sizes seem bigger than Mt. Everest, do not lose hope fellow developer!!

We have a few ways for you to get your bundles from fat to fit in no time! :p

## 1. Analyse your bundle
The first step to solving a problem is to understand it, i.e., look at what is making your bundles so big.

There are a few bundle analysers out there, but I chose [`source-map-explorer`](https://github.com/danvk/source-map-explorer#readme) as it seemed to get the job done nicely, and it generates an HTML file for us to see all the details of our bundle, especially all the dependencies and sub-dependencies

Run the following command after navigating to your React project in the terminal:

```
npx source-map-explorer {path to bundle}/bundle.js {path to bundle}/bundle.js.map
```

This builds the bundle, downloads the `source-map-explorer` npm package and displays the sizes of all the dependencies and sub-dependencies.

## 2. Extract junk files from your bundle
One observation I had was that there were a lot of junk (`.svg`, `.eot`, `.ttf`, ...) files being included in the bundle, which were never supposed to show up in a JS bundle (duh). 

So, to remove these junk files, I modified the webpack config to load all such incoming files into their resp. directories, as shown in this code snippet: 

```diff
... some webpack config settings
  module: {
    rules: [
      ... some loader settings
      {
        test: /\.jpg$/,
        loader: 'file-loader',
      },
      {
        test: /\.(png|jpg|jpeg|gif)$/,
        loader: 'url-loader',
      },
      {
        test: /\.svg$/,
-       loader: 'url-loader',
+       loader: 'file-loader?name=assets/images/[name].[ext]',
      },
      {
        test: /\.eot/,
-       loader: 'url-loader',
+       loader: 'file-loader?name=assets/fonts/[name].[ext]'
      },
      {
        test: /\.ttf/,
-       loader: 'url-loader',
+       loader: 'file-loader?name=assets/fonts/[name].[ext]&mimetype=application/x-font-ttf'
      },
      {
        test: /\.woff/,
-       loader: 'url-loader',
+       loader: 'file-loader?name=assets/fonts/[name].[ext]&mimetype=application/font-woff'
      },
      {
        test: /\.woff2/,
-       loader: 'url-loader',
+       loader: 'file-loader?name=assets/fonts/[name].[ext]&mimetype=application/font-woff2'
      }
      ... some more loader settings
    ]
  },
... some more webpack config rules
```

This alone reduced bundle sizes by >=2MB! :o

## 3. Extract CSS/LESS into a CSS file (this might break some CSS)
Next, we found that a lot of CSS/LESS code was being injected into the bundle as internal styles (via `<style />` tags in the browser DOM), which seems like a lot of unnecessary overhead for the browser. 

So, to remove CSS/LESS from JS, we extracted the CSS/LESS code into a separate file, called `main.css` and included this file in our HTML file, like so:

In your Webpack config (`webpack.config.js`) file: 

```diff
... some webpack config settings
  module: {
    rules: [
      ... some loader settings
      {
        test: /\.(css|less)$/,
        loader: [
          {
-           loader: 'style-loader',
-           query: {
-             sourceMap: true,
-           },
+           loader: MiniCssExtractPlugin.loader,
+           options: {
+             hmr: process.env.NODE_ENV === 'development',
+           },
          },
          {
            loader: 'css-loader'
          },
          {
            loader: 'less-loader',
            options: {
              paths: [
                path.resolve(__dirname),
                path.resolve(__dirname, 'node_modules'),
              ],
            },
          },
        ],
      },
      ... some more loader settings
    ]
  },
... some more webpack config rules
```

In your main HTML file (like, `index.html`): 

```diff
<!DOCTYPE html>
  <head>
    ...
+   <link rel='stylesheet' href='{path to public dir}/main.css'></link>
  </head>
</html>
```

This could potentially reduce bundle size by ~1MB

### 4. Ditch moment.js (seriously, it sucks)

Ditch moment.js. Seriously.

We replaced `moment.js` with [`dayjs`](https://github.com/iamkun/dayjs/blob/dev/docs/en/API-reference.md), which has a similar API and a plugin [`dayjs-ext`](https://github.com/prantlf/dayjs#plugin) that supports timezone-based date display, which we needed to display dates in the `PDT`, as per our use case. 

However, this package itself imports a huge sub-dependency (`timezone-support`).

To reduce its impact, we discovered that there are [multiple versions](https://github.com/prantlf/dayjs/blob/combined/docs/en/Plugin.md#timezone) of `dayjs-ext` available with various sizes, based on year ranges you want to display the dates for. 

So, we wrote some code into our webpack config, to use the smallest possible version of the dependency, based on the current year at the time of starting the bundling.

In your `package.json`:

```diff
-"moment-timezone": "^0.5.23",
+"dayjs": "^1.8.13",
+"dayjs-ext": "^2.2.0",
```

In your Webpack Config (`webpack.config.js`), do:

```diff
... some webpack config settings
  resolve: {
    ... some settings
+    new webpack.IgnorePlugin({
+      resourceRegExp: /timezone-support\/dist\/data$/,
+    }),
+    new webpack.NormalModuleReplacementPlugin(
+      /dayjs-ext\/plugin\/timeZone/,
+      function(resource) {
+        let BEST_DATA_STR = 'timeZone'
+        const CURRENT_YEAR = new Date(Date.now()).getFullYear()
+
+        if (CURRENT_YEAR <= 2022) BEST_DATA_STR = 'timeZone-2012-2022'
+        else if (CURRENT_YEAR <= 2038) BEST_DATA_STR = 'timeZone-1970-2038'
+        else if (CURRENT_YEAR <= 2050) BEST_DATA_STR = 'timeZone-1900-2050'
+
+        resource.request = resource.request.replace(/timeZone/, BEST_DATA_STR)
+      },
+    ),
    ... some more settings
  },
... some more webpack config rules
```

Replace your `moment` imports with `dayjs` imports:

```diff
-import moment from 'moment-timezone'
+import dayjs from 'dayjs'
+import dayjsTimezonePlugin from 'dayjs-ext/plugin/timeZone'
+dayjs.extend(dayjsTimezonePlugin)

...
- const formattedDate = moment(timestamp).tz(timezone).format(format)
+ const formattedDate = dayjs(timestamp).format(format, { timeZone: timezone })
...
```

This will reduce your bundle size by ~1MB

### 5. Import differently (Tree shaking)
Now, the difficult part starts, removing JS from our JS bundles. Not to worry, we can make our bundles small again.

In some libraries, like `semantic-ui-react` and `lodash`, when we do named imports 
(the imports with the curly brackets, that look like `import { Accordion } from 'semantic-ui-react'`), we end up importing the entire library, instead of just the components we use.

That's because imports like this work by importing the entire package, and then extracting the specified pieces into constants (like, `Accordion`)

To prevent importing the entire library just for a couple of functions/components/whatever, we need to import only what we need. To do this, we have to change the way we import some libraries.

> This only works, if the library/framework exports its components/functions in the given manner. If it doesn't, sorry, but you seem to be unlucky

>> The code snippet below shows how to implement it for `semantic-ui-react`, but you can apply this to any major JS library that exports its functions in this format, like `lodash` [does](https://stackoverflow.com/questions/35250500/correct-way-to-import-lodash)

```diff
-import { Accordion } from 'semantic-ui-react';
+import Accordion from 'semantic-ui-react/dist/es/modules/Accordion/Accordion';
```

This tactic has varying results, implementing this for `semantic-ui-react` reduces the bundle size by ~300KB, but results may vary with the library you use this technique for

## 6. Innovate! (and update this doc pls)
This is by no means a comprehensive guide, and needs to evolve with our components and use cases. 
Hopefully, this guide has armed you with the necessary knowledge and tools to fight the epidemic of huge bundle sizes.

If you find any good tools/tips/ideas that have not been covered in this guide, please feel free to send a PR to this repo with your suggestions, we would be happy to share your learnings with everyone!