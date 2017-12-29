* Start Date: 2017-12-26
* RFC PR: (leave this empty)
* React Issue: (leave this empty)

# Summary

A route prefix plugin for FusionJS.

# Motivation

For web apps not deployed to the root of a host domain, the app must today be aware of the host routing to the app itself.  For example, the web app powering 'https://www.uber.com/drive/' may have routes for '/safety' and '/partner-app'.  Today, these routes are tightly coupled to the host routing and would therefore have to be registered as '/drive/safety' and '/drive/partner-app'.

Providing the capability to configure a route prefix facilitates decoupling these concerns, allowing a web app to more easily be deployed without concern for the host path.  As an example, configuration of a route prefix would allow for the registering '/safety' the work seamlessly for both 'https://drive.uber.com/safety' and 'https://uber.com/drive/safety'.

# Detailed design

The impementation involves three main concerns:

* Static asset pathing
* `ctx.url` re-write
* Prefix handling for `fusion-plugin-reactroute`

### Static asset pathing

`fusion-cli` currently handles supplying the static asset path via the environment variables plugin.  Values are supplied via [process.env](https://nodejs.org/api/process.html#process_process_env) with reasonable fallback defaults.

For example, the static asset pathing is defined by the key, `FRAMEWORK_STATIC_ASSET_PATH`, with a default value of `/_static`.

We can provide the route prefix in a similar fashion, and update static asset path resolution accordingly:

* Passing `ROUTE_PREFIX` as an environment variable that can be accessed via the environment variables plugin.
* Ensure that the default static asset path incorporates the route prefix (e.g. '/privacy/_static', where '/privacy' is the route prefix).

In production, this will likely be supplied via an app's `pinocchio.yaml` configuration file:

```
website:
  command: 'ROUTE_PREFIX="/privacy" FRAMEWORK_STATIC_ASSET_PATH="https://someuuid.cloudfront.net/uberprivacy" npm start'
```

Ideally, as this is being provided as an environment variable, we should scope this to the `fusion-cli` and ensure it is not exposed to user land.

Note that the primary [webpack considerations](https://github.com/fusionjs/fusion-cli/search?utf8=%E2%9C%93&q=__webpack_public_path__&type=) here are already handled via the [`compilation-metadata-plugin`](https://github.com/fusionjs/fusion-cli/blob/master/plugins/compilation-metadata-plugin.js) plugin.

### `ctx.url` rewrite

Many middleware are expected to run conditional checks against `ctx.url`.  It is not the concern of the middleware to know whether the web app is deployed at the root of a host, or behind a route prefix.  In order to enforce this, a core middleware can strip the route prefix from `ctx.url` prior to any of the user-land middlewares' execution.  This has the benefit of allowing middlewares to be easily re-used across applications regardless of deployment concerns.

e.g.
```
# fusion-cli: src/route-prefix-plugin.js
const envVarsPlugin = require('./environment-variables-plugin');
module.exports = function() {
  const envVars = envVarsPlugin().of();
  return function middleware(ctx, next) {
    const prefix = envVars.prefix;

    // enhance ctx.url sans prefix
    if(ctx.url.indexOf(prefix) === 0 /*found at index 0*/) {
      ctx.url = ctx.url.slice(prefix.length);
    }
  }
}
```

### Prefix handling for `fusion-plugin-react-router`

In order to ensure that the router works as expected, we will want to pass in the prefix as part of the [configuration](https://github.com/fusionjs/fusion-plugin-react-router#router).

Today, the router has the option to take in a `basepath` as a prop:

```
import {Router} from 'fusion-plugin-react-router';

<Router
  location={...}
  basename={...}
  context={...}
  onRoute={...}
>{child}</Router>
```
[(link)](https://github.com/fusionjs/fusion-plugin-react-router#router)

As user land plugins should not be accessing environment variables directly, this will likely be supplied by a user land `{config}.js` file.  This file must remain synchronized to the configuration used in the static asset plugin and the ctx re-write.  Due to this being sensitive to errors, it may be prudent to have a build time check here to ensure synchronization.

# Drawbacks

This is a non-trival amount of effort to implement.  In its current form, it is also prone to user error and requires duplicated configuration.  As noted below, the alternative in the [dependency injection](https://github.com/fusionjs/rfcs/pull/1/commits/8c440259291c9774b44b9dca07cf35dbf2a5f98e?short_path=9919d4f#diff-9919d4f45f05666e94a1bcb5edd7f51a) world is potentially cleaner.  Implementing the proposal above will require a migration path towards the DI implementation down the road.

# Alternatives

Alternatively, the open proposal for migrating towards a purer [dependency injection](https://github.com/fusionjs/rfcs/pull/1/commits/8c440259291c9774b44b9dca07cf35dbf2a5f98e?short_path=9919d4f#diff-9919d4f45f05666e94a1bcb5edd7f51a) system for the framework may yield a cleaner solution here as well.  At a high level, the solution may involve the following concerns:

* Creating a `RoutePrefixPluginConfigToken` token to use for dependency injection.
** Configuration object would include the route prefix.
* Registering both the `RoutePrefixPlugin` and its config (with a placeholder configuration - for example, `/`) outside of user land.
* Allow users to override the default config by registering a new configuration with the same `RoutePrefixPluginConfigToken` token.

This allows a single configuration object to be used across the core plugins and user land, eliminating the need for manual synchronization.

# Unresolved questions

* Do we want a build-time check to ensure that duplicated configuration files are synchronized?
* Do we want a scaffold-time question to populate the route prefix (i.e. create a `{config}.js` object and/or populate `pinocchio.yaml`)
* In terms of timeline, do we want to implement a pre-DI solution?
* Would the proposed solution live in `fusion-cli` or `fusion-core`?  If `fusion-core`, would we want to migrate plugins in `fusion-cli` to core as part of this refactor?
* Which existing plugins will need to be modified to include support for a route prefix?
** For example, `fusion-plugin-font-loader-react` has hard-coded '[/_static](https://github.com/fusionjs/fusion-plugin-font-loader-react/blob/e50ea98440adcba0d4e733e641d564fb2da09fca/src/generate-font-faces.js)' paths today.
* What is the best way to handle the HMR case for webpack?  Today, `__webpack_public_path__` does not appear to be respected.
