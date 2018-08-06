* Start Date: 2018-08-06
* RFC PR: (leave this empty)
* Fusion Issue: (leave this empty)

# Summary

An API to get an ID that is the same across server and browser.

# Basic example

```js
import {createUniversalId} from "fusion-core";

const id1 = createUniversalId();
// id1 will be the same exact value in server and browser

const id2 = createUniversalId();
// id2 will be the same exact value in server and browser

// id1 and id2 are guaranteed to be different from each other
```

# Motivation

When implementing plugins, there is often a need for some consistent key between server and client.

One concrete use case is when Fusion plugins pass data from the server to the client. This process typically involves some form of the following steps:

1. On the server, serialize data into some payload
2. On the server, insert this payload into page somehow
3. On the client, extract this payload from the page somehow
4. On the client, deserialize this data from the payload

Some form of hardcoded string key is used for this purpose, but this approach has a couple problems:
- "Manually" defined strings are prone to collisions.
- Generated IDs would be better, but then there's no guarantee of sameness across both server and browser.

# Detailed design

This is ideally implemented as a [Fusion compiler plugin](https://github.com/fusionjs/rfcs/pull/7).

## Build-time macro
(before)
```js
const a = createUniversalId();
const b = createUniversalId();
```

(after)
```js
const a = "__fusion_uid_0";
const b = "__fusion_uid_1";
```

## Build-time constraints

The macro must not be invoked inside a function. This can be enforced with a build-time error.

```js
import {createUniversalId} from "fusion-core";

function attemptedRuntimeId() {
  // NOPE!
  // This should yield a build-time error.
  return createUniversalId();
}

createUniversalId() // This is OK
```

# Drawbacks

- String collisions are theoretically possible. Instead, the likelyhood is simply reduced via a shared "generator" that guarantees each returned value is unique.

# Alternatives

## Symbols

One alternative possibility is exposing an API for consistent symbols (using the global symbol registry).

(before)
```js
import {createUniversalSymbol} from "fusion-core";

const a = createUniversalSymbol();
const b = createUniversalSymbol();
```

(after)
```js
const a = Symbol.for("__fusion_uid_0");
const b = Symbol.for("__fusion_uid_1");
```

- Symbol serialization would be `Symbol.keyFor(symbol)`
- Symbol deserialization is just `Symbol.for(key)`

However, using actual symbols by default have a couple drawbacks:
1. Compared to strings, they are less convenient when using `JSON.stringify` and `JSON.parse`.
2. Symbol usage in old browsers requires expensive polyfills.

Furthermore, symbols can be easily created using the lower-level string ID API:

```js
import {createUniversalId} from "fusion-core";

const symbol = Symbol.for(createUniversalId());
```

# Adoption strategy

This is an entirely new API, but it dovetails nicely with proposals such as https://github.com/fusionjs/rfcs/pull/8.

# How we teach this

We should write documentation that explains how to use this feature instead of using hardcoded strings.

# Unresolved questions

TBD
