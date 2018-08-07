* Start Date: 2018-07-24
* RFC PR: (leave this empty)
* Fusion Issue: (leave this empty)

# Summary

Fusion plugins should be able to perform operations at build-time.

# Basic example

```js
// src/app.js

import {foo} from "fusion-plugin-foo";

foo();
```

```json
{
  "name": "fusion-plugin-foo",
  "...": "...",
  "fusion": {
    "compiler-plugin": "./compiler-plugin.js"
  }
}
```

```js
// node_modules/fusion-plugin-foo/compiler-plugin.js
module.exports = function plugin() {
  return {
    analyzer: (...babelPluginArgs) => {
      // Traverse AST to count number of times `foo` is invoked
      return {callCount};
    },
    transformer: (...babelPluginArgs) => {
      // Conceptually equivalent to "babel-macros" idea
      // Statically replace `foo()` CallExpression with "foo" StringLiteral
    },
    aggregator: (allMetadata, compiler) => {
      let count = 0;
      for (let [filename, metadata] of allMetadata.entries()) {
        if (metadata.callCount) {
          count += metadata.callCount;
        }
      }

      const codeForServerEntry = `
        console.log("foo() was invoked ${callCount} times");
        `;

      return {server: codeForServerEntry};
    },
  };
};
```

# Motivation

Many fusion plugins rely on static analysis, buildtime transformations, and code generation including:

- `fusion-plugin-i18n`
- `fusion-react-async`
- Future plugins such as experimentation

However, the build-time functionality is currently implemented in `fusion-cli` which is far from ideal.

# Detailed design

## High-level compiler phases

1. Analysis
   - Only analysis/metadata extraction, no modification of AST
2. Macros
   - Macros on source code, (e.g. JSX to JSX macros)
4. ECMAScriptification
   - Transforms user source code into standard ECMAScript that can be parsed by webpack (e.g. JSX/Flow transforms)
5. Optimization
   - Bundling
   - Minification

## Compiler plugins

Compiler plugins are composed of three high-level functions:

### Analyser

Returns serializable metadata given a file's AST. This must be serializable because this step might be cached. Run in analysis phase.

- Useful for i18n, etc.

### Transformer

Transforms a file's AST. Might be cached. This is equilavent to the idea of "babel macros". Run in macro phase.

### Aggregator

Consumes aggregated metadata. Returns JS source code to be injected into entry. Run in bundle phase.

## Implementation

### Webpack `babel-loader` equivalent

1. Parse file source into AST
2. Extract import statements from AST
3. Use webpack loader [`resolve` API](https://webpack.js.org/api/loaders/#this-resolve) to find package.json metadata associated with imports
4. Collect plugins from "fusion-compiler-plugin" field from package.json metadata
5. Run plugins


Benefits of this strategy:
- No additional file I/O overhead, since imported dependencies need to be resolved by webpack anyways.
- No additional parse overhead, since these files must be parsed by Babel anyways.

### Post-bundling transpilation

ES2018 to ES5 transpilation will be performed after bundling.

# Drawbacks

**Exposing Babel as part of public API**

This might make migrating to a different compiler tools more difficult later.

# Alternatives

**`babel-plugin-macros`**

Macros alone are insufficient for many use cases; often some form of program-wide static analysis and code-generation is required.

# Adoption strategy

- The initial rollout can be a pure refactor with no code changes for Fusion.js users (only dependency upgrades)
- Worthwhile possible future explorations include:
  - Extracting this into a standalone Webpack plugin eventually
  - Merging with `babel-plugin-macros`
  - Collaborating with "single-parse" bundlers/compilers such as Parcel to support similar functionality out of the box

# How we teach this

The word "plugin" is already used a lot in Fusion, so perhaps different terminology might be better.

# Unresolved questions

- Should minification or transpilation happen first?
- What about browser or server-only transformers?
- Is it sufficient if transformers/analyzers only run on files that import the package containing the compiler plugin?
