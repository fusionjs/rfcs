* Start Date: 2018-02-01
* RFC PR: (leave this empty)
* React Issue: (leave this empty)

# Summary

Currently, fusion-cli will generate an ES5 bundle for browsers. Ideally, browsers get a bundle with the minimum possible features transpiled.

# Motivation

Always serving ES5 bundles for browsers that support ES2015+ is suboptimal, both in terms of runtime performance and bundle size. We can achieve a more optimal outcome by serving minimally transpiled code.

Another benefit is presumably development is done in a modern browser, so the opportunity to skip transpilation entirely should improve incremental build times.

A possible implementation strategy (described in the following section) would also involve whole-program analysis, rather than the piecemeal, file-based `babel-loader` transpilation approach that is currently used. This change might make transpilation-related, program-level optimizations, such as polyfilling and helper deduplication easier to implement.

# Detailed design

The key idea is to transpile code *after* bundling.

### Lazy transpilation

In development, the general process would go something like this:

0. Build is completed, creating an untranspiled bundle (`0-contenthash-raw.js`, `1-contenthash.raw.js`, etc.)
1. Request from browser is recieved
2. UA is parsed to determine browser information
3. Browser information is passed to `babel-preset-env`, generating a list of required transforms/polyfills
4. A unique hash is generated from this list
5. The server uses this hash to construct the chunk paths (`0-contenthash-featurehash.js`, etc.) in the manifest, and response is sent
6. When recieving a request for a chunk
   - If the corresponding bundle already exists, respond immediately
   - Otherwise, block the request and transpile the raw bundle and write it to disk. Respond when transpilation is complete.

### Production builds
To avoid runtime transpilation in production, the lazy process could be run at build-time (given an exhaustive set of supported browsers versions). For any browsers encountered at runtime with a feature hash not covered at buildtime, fallback to the ES5 bundle.

# Drawbacks

### Complexity
This would add complexity to asset/chunk logic.

### Incremental transpilation
By running Babel after bundling, we lose the incremental transpilation provided by `babel-loader`, where only *changed* files are re-transpiled. With the new approach, the whole bundle will need to be re-transpiled when changed.

Fortunately, there are a two mitigating factors to this problem:
1. The use of bundle splitting will provide some granularity to the bundles, so only changed chunks will need to be re-transpiled, not the entire bundle.
2. Incremental transpilation only matters in development, where modern browsers are typically used. In the case of Chrome, transpilation may not be needed at all!

### Double Babel phases
We currently use Babel for a number of things that aren't strictly ECMAScript transpilation (JSX, flow annotation removal, macros, etc.), so the existing `babel-loader` phase would not necessarily go away. Fortunately, these use cases can probably be implemented differently, thereby eliminating this. Using Parcel (which has a single parse for both bundling and babel plugins) would also reduce the marginal cost.

### Unreliability of browser user agent headers
Relying on the user agent header will not be reliable 100% of the time. To guard against this, there needs to be some error handling  logic to fallback to the ES5 bundle in case there is a script parse error.

# Alternatives

### Multiple, broad browser-category builds
One alternative is to produce a few builds for specific, broad categories of browsers (e.g. ES5/legacy, Evergreen/ES2017+)
This is essentially a less generalized solution of this RFC. Implementing this would involve the same underlying components and probably not save much implementation complexity.

# Adoption strategy

This is a largely under-the-hood change with little to no developer-facing consequences. A minor version bump to fusion-cli should be sufficient.

# How we teach this

Documenting the differences between development and production will be important (particularly if production browser support ranges are configurable).

# Unresolved questions

### Source maps
Running Babel *after* bundling is much less common, so it must be verified that source mapping still works perfectly.
