There are a lot of exciting things happening in the WebAssembly world. This
tutorial is intended for React developers who are interested in adding some
WebAssembly to their apps, using the Rust language. I'm going to assume you
already know what WebAssembly and Rust are and why you might want to do this.

I will cover only the basics, and try to keep the tooling and complexity to a
minimum.

# Source

Complete source code for the final running example is available
[on GitHub](https://github.com/anderspitman/react_rust_wasm)

# Prerequisites

You'll first need to have Rust and node installed. They both have excellent
installation documentation:

* [Rust and cargo](https://www.rust-lang.org/en-US/install.html)
* [node and npm](https://nodejs.org/en/)

# Create the React App

We'll start with a barebones React app. First, create the directory
`react_rust_wasm`, and cd into it.

Create the following directories:

```
src
build
dist
```

Then, initialize the npm package with
default options:

```bash
npm init -y
```

Next, install React, Babel, and Webpack:

```bash
npm install --save react react-dom
npm install --save-dev babel-core babel-loader babel-preset-env babel-preset-react webpack webpack-cli
```

Then, create the following source files:

`dist/index.html`:
```html
<!doctype html>
<html>
  <head>
    <meta content="text/html;charset=utf-8" http-equiv="Content-Type"/>
    <title>React, Rust, and WebAssembly Tutorial</title>
  </head>
  <body>
    <div id="root"></div>
    <script src="/bundle.js"></script>
  </body>
</html>
```

`src/index.js`:
```javascript
import React from 'react';
import ReactDOM from 'react-dom';

ReactDOM.render(
  <h1>Hi there</h1>,
  document.getElementById('root')
);
```

We will also need a `.babelrc` file:

```json
{
  "presets": [
    "react",
    "env",
  ],
}
```

And a `webpack.config.js` file:

```javascript
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'bundle.js',
    path: path.resolve(__dirname, 'dist')
  },
  module: {
    rules: [
      {
        test: /\.(js|jsx)$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader',
        }
      }
    ]
  },
  mode: 'development'
};
```

You should now be able to test that the React app is working. Run:

```bash
npx webpack
```

This will generate `dist/bundle.js`. If you start a web server in the `dist`
directory you should be able to successfully serve the example content.

At this point we have a pretty minimal working React app. Let's add a button
so we have a little interaction. We'll use the button to activate a dummy
function that represents some expensive computation, which we want to
eventually replace with Rust/wasm for better performance. Replace
`src/index.js` with the following:

```javascript
import React from 'react';
import ReactDOM from 'react-dom';

function bigComputation() {
  alert("Big computation in JS");
}

const App = () => {
  return (
    <div>
      <h1>Hi there</h1>
      <button onClick={bigComputation}>Run Computation</button>
    </div>
  );
};

ReactDOM.render(
  <App />,
  document.getElementById('root')
);
```

Now if you should get an alert popup when you click the button, with a
message indicating that the "computation" is happening in JavaScript.


# Adding a Splash of Rusty WASM

Now things get interesting. In order to compile Rust to WebAssembly, we
need to configure a few things.

## WebAssembly Dependencies

First, we need to use Rust nightly. You can switch your Rust toolchain to
nightly using the following command:

```bash
rustup default nightly
```

Next, we need to install the necessary tools for wasm:

```bash
rustup target add wasm32-unknown-unknown
cargo install wasm-bindgen-cli
```

## Create the Rust project

In order to build the Rust code, we need to add a `Cargo.toml` file with the
following content:

```toml
[package]
name = "react_rust_wasm"
version = "1.0.0"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

You can ignore the lib section for this tutorial. Note that we have
`wasm-bindgen` in the dependencies section. This is the Rust library that
provides all the magic that makes communicating between Rust and JavaScript
possible and almost painless.

Now create the source file `src/lib.rs` to contain our Rust code:

```rust
#![feature(proc_macro, wasm_custom_section, wasm_import_module)]

extern crate wasm_bindgen;

use wasm_bindgen::prelude::*;

#[wasm_bindgen]
extern {
    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn big_computation() {
    alert("Big computation in Rust");
}
```

For the purposes of this tutorial, it's not too important to understand
everything that's going on here. Basically, decorating Rust code with
`wasm_bindgen` bridges that piece of code to JavaScript. In this case,
the extern section is making the JS `alert` function available 
in our Rust code, while `big_computation` is being made available
to be called from JS.

## Building

We're now ready to build everything. There are a couple stages to this. We're
going to implement these as simple npm scripts. Of course there are lots of
fancier ways to do this.

The first stage is to compile the Rust code into wasm. Add the following to
your package.json scripts section:

```json
"build-wasm": "cargo build --target wasm32-unknown-unknown"
```

If you run `npm run build-wasm`, you should see that the file
`target/wasm32-unknown-unknown/debug/react_rust_wasm.wasm` has been created.

Next we need to take the wasm file, and convert it into the final form that
can be consumed by JavaScript, in addition to generating the proper JS files
for wrapping everything. Add the following script to package.json:

```json
"build-bindgen": "wasm-bindgen target/wasm32-unknown-unknown/debug/react_rust_wasm.wasm --out-dir build"
```

If you run `npm run build-bindgen`, you should see several files created in
the build directory.

Note that `wasm-bindgen` even creates a react_rust_wasm.d.ts file for you
in case you want to use TypeScript. Nice!

Ok, now all we need is a build script to do all the steps in order:

```json
"build": "npm run build-wasm && npm run build-bindgen && npx webpack"
```

Your package.json scripts section should now look something like this:

```json
"scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "build-wasm": "cargo build --target wasm32-unknown-unknown",
    "build-bindgen": "wasm-bindgen target/wasm32-unknown-unknown/debug/react_rust_wasm.wasm --out-dir build",
    "build": "npm run build-wasm && npm run build-bindgen && npx webpack"
  },
```

Running `npm run build` should work at this point. However, we still need to
modify our JavaScript code to use our wasm module instead of the JS function.

Replace `src/index.js` with the following:

```javascript
import React from 'react';
import ReactDOM from 'react-dom';

const wasm = import("../build/react_rust_wasm");

wasm.then(wasm => {

  const App = () => {
    return (
      <div>
        <h1>Hi there</h1>
        <button onClick={wasm.big_computation}>Run Computation</button>
      </div>
    );
  }

  ReactDOM.render(
    <App />,
    document.getElementById('root')
  );
});
```

There are a couple important changes. First, as of this writing, you need to
use the `import` function, rather than the normal ES6 import syntax. It has
something to do with not being able to load wasm asynchronously yet. In order
to use this function, we need to enable a babel plugin. Install it with the
following:

```bash
npm install --save-dev babel-plugin-syntax-dynamic-import
```

And add it to your .babelrc:

```json
{
  "presets": [
    "react",
    "env",
  ],
  "plugins": ["syntax-dynamic-import"]
}
```

The `import` function returns a promise. That's why we need to call
`wasm.then` in order to kick things off.

You should now be able to successfully run `npm run build`. Reload
`dist/index.html` from a web server and you'll now see a message indicating
it's running from Rust. And just like that, we're done!

# Where to go from here

There are a lot of exciting things happening in the world of Rust+WebAssembly.
This tutorial was aimed at React developers who just want to get their feet
wet. Here are a few other resources you can check out if you want to go
deeper.

* [rustwasm/team](https://github.com/rustwasm/team). This seems to be the
central repository for keeping up with the current state of Rust and WebAssembly.
It's a fantastic resource.

* [wasm-bindgen](https://github.com/rustwasm/wasm-bindgen) I highly recommend
reading through their documention and examples. A good chunk of this tutorial is copied
almost exactly from there. There are many more advanced features that can be
used, such as using other JavaScript APIs, defining structs in Rust and using
them in JS, passing those structs between Rust and JS, and many more.

* [stdweb](https://github.com/koute/stdweb) is a bridging library that has
some overlap with `wasm-bindgen`. `stdweb`
has some nice features and macros for letting you write JavaScript inline in
your Rust code, rather than just a simple bridge. `wasm-bindgen` seems to be
more focused on bridging, and is designed to be used with languages other than
just Rust in the future.

* [Yew](https://github.com/DenisKolodin/yew) is a Rust framework for writing
client-side apps. It's heavily inspired by React, but it lets you write your
app 100% in Rust. 
