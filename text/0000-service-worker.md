* Start Date: 2018-08-08
* RFC PR: (leave this empty)
* Fusion Issue: (leave this empty)

# Summary

Service worker support in Fusion

# Basic example

Most developers will want to use a default service worker implementation (as defined in the SW plugin) without modification. This is how it is used in this case:

Note: Fusion CLI is hardcoded to look for a `src/sw-handlers.js` file (akin to how it uses `src/main.js`).
```js
// src/sw-handlers.js

import {serviceWorkerLogic} from "fusion-plugin-sw";

export default serviceWorkerLogic;
```

Add registration code to app:
```js
// src/main.js
import {ServiceWorkerPlugin} from "fusion-plugin-sw";

export default () => {
  const app = new App(/* ... */);
  if (!__DEV__) {
    app.register(ServiceWorkerPlugin);
  }
  // ...
  return app;
}

```

# Motivation

<!--
Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.
-->

# Detailed design

Two low-level APIs:

1) New virtual module `serviceWorkerTemplate` in public API:

```js
import {serviceWorkerTemplate} from "fusion-core";

const serializableParam = {
  criticalScripts: [/* ... */],
  staticAssets: [/* ... */],
};

const swSourceCode = serviceWorkerTemplate(serializableParam);
// swSourceCode is a string containing a JS program/bundle that invokes the default exported function from "src/sw-handlers.js" with `serializableParam`
```

2) New entry file: `src/sw-handlers.js`:
```js
// src/sw-handlers.js

export default (deserializedParam) => {
  // deserializedParam is equivalent to `JSON.parse(JSON.stringify(serializableParam))`

  // Arbitrary logic goes here...
  self.addEventListener("install", /* ... */);
  self.addEventListener("fetch", /* ... */);
  // ...
}
```

<!--
This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with Fusion to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.
-->

# Drawbacks

<!--
Why should we _not_ do this? Please consider:

* implementation cost, both in term of code size and complexity
* whether the proposed feature can be implemented in user space
* the impact on teaching people Fusion
* integration of this feature with other existing and planned features
* cost of migrating existing Fusion applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.
-->

# Alternatives

<!--
What other designs have been considered? What is the impact of not doing this?
-->

# Adoption strategy

<!--
If we implement this proposal, how will existing Fusion developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?
-->

# How we teach this

<!--
What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Fusion patterns?

Would the acceptance of this proposal mean the Fusion documentation must be
re-organized or altered? Does it change how Fusion is taught to new developers
at any level?

How should this feature be taught to existing Fusion developers?
-->

# Unresolved questions

- What is the exact API of the function exported in `src/sw-init.js`? Or in other words, which paramaters should it accept?
- Is there a better name than `src/sw-init.js`?
