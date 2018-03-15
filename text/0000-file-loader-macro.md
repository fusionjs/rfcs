* Start Date: 2018-03-15
* RFC PR: (leave this empty)

# Summary

Expose a `assetContents` macro (similar to `assetUrl`) that resolves to the contents of a file as a string at compile-time.

# Basic example

`some/file.txt`

```txt
hello world
```

```js
import {assetContents} from 'fusion-core';

console.log(assetContents('some/file.txt')); // hello world
```

# Motivation

There needs to be an elegant way of importing files as strings at compile time. Currently, one needs to do it at runtime, and this involves a lot of boilerplate if the file contents are only used client-side.

# Detailed design

This would likely be implemented with a Babel plugin, but would still not expose loaders as a API from Fusion.js.

The plugin would simply replace a CallExpression with a `assetContents` caller identifier with a string literal corresponding to the contents of the file specified by the string literal argument.

# Drawbacks

There may be confusions about the fact that `assetContents` will only work with a string literal.

There may also be performance implications if this tool is abused.

# Alternatives

The most common suggestion is to allow arbitrary webpack loaders, so that the raw loader can be used. This has a few problems: the webpack configuration for Fusion.js is extremely complex and opening up arbitrary extensibility can make it difficult to make changes to it. For example, one recent effort involved an upgrade to webpack 4. If webpack config was exposed, migrating an app that extended webpack config would likely be far more difficult.

Another alternative is to load files at runtime via `fs.readFile` and `ctx.template.body.push`. This works but it tends to require a fairly solid understanding of Fusion.js' asynchrony model and plugin architecture.

# Adoption strategy

This would be a feature addition. Simply updating to the latest minor version of fusion-cli would be enough to gain access to this feature. Users using workarounds such as forking fusion-cli or the workaround mentioned in the previous section could migrate at their convenience or only use the new feature for new development without risk of impact to legacy code.

# How we teach this

The documentation can draw parallels with `assetUrl`, since both work similarly.

# Unresolved questions

What about other similar cases? For example, CSS loading, SVG loading as component.

It may make sense to think about other loader-related usage patterns to arrive at a more generic solution.
