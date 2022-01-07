# ember-simple-auth-oidc

[![npm version](https://badge.fury.io/js/ember-simple-auth-oidc.svg)](https://www.npmjs.com/package/ember-simple-auth-oidc)
[![Test](https://github.com/adfinis-sygroup/ember-simple-auth-oidc/workflows/Test/badge.svg?branch=master)](https://github.com/adfinis-sygroup/ember-simple-auth-oidc/actions?query=workflow%3ATest)
[![Code Style: Prettier](https://img.shields.io/badge/code_style-prettier-ff69b4.svg)](https://github.com/prettier/prettier)
[![semantic-release](https://img.shields.io/badge/%20%20%F0%9F%93%A6%F0%9F%9A%80-semantic--release-e10079.svg)](https://github.com/semantic-release/semantic-release)

A [Ember Simple Auth](http://ember-simple-auth.com) addon which implements the
OpenID Connect [Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth).

## Installation

* Ember.js v3.24 or above
* Ember CLI v3.24 or above
* Node.js v12 or above

Note: The addon uses [Proxy](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) 
in its implementation, if IE browser support is necessary, a polyfill needs to be provided.

```bash
$ ember install ember-simple-auth-oidc
```

## Usage
To use the oidc authorization code flow the following elements need to be added
to the Ember application.

The login / authentication route (for example the Ember Simple Auth default `/login`)
needs to extend from the `OIDCAuthenticationRoute`, which handles the authentication
procedure. In case the user is already authenticated, the transition is aborted.

```js
// app/routes/login.js

import OIDCAuthenticationRoute from "ember-simple-auth-oidc/routes/oidc-authentication";

export default class LoginRoute extends OIDCAuthenticationRoute {}
```

Authenticated routes need to call `session.requireAuthentication` in their 
respective `beforeModel`, to ensure that unauthenticated transitions are 
prevented and redirected to the authentication route.

```js
// app/routes/protected.js

import Route from "@ember/routing/route";
import { inject as service } from "@ember/service";

export default class ProtectedRoute extends Route {
  @service session;

  beforeModel(transition) {
    this.session.requireAuthentication(transition, "login");
  }
}
```

To include authorization info in all Ember Data requests override `headers` in
the application adapter and include `session.headers` alongside any other
necessary headers. By extending the application adapter from the `OIDCAdapter`,
the `access_token` is refreshed before Ember Data requests, if necessary. The
`OIDCAdapter` also provides default headers with the authorization header
included.

```js
// app/adapters/application.js

import { inject as service } from "@ember/service";
import OIDCAdapter from "ember-simple-auth-oidc/adapters/oidc-adapter";

export default class ApplicationAdapter extends OIDCAdapter {
  @service session;

  get headers() {
    return { ...this.session.headers, "Content-Language": "en-us" };
  }
}
```

The `OIDCAdapter` already handles unauthorized requests and performs an 
invalidation of the session which also remembers your visited URL. If you want 
this behaviour for other request services as well, you can use the
`handleUnauthorized` function. The following snippet shows an example
`ember-apollo-client` afterware (error handling) implementation:

```js
// app/services/apollo.js

import { inject as service } from "@ember/service";
import { onError } from "apollo-link-error";
import ApolloService from "ember-apollo-client/services/apollo";
import { handleUnauthorized } from "ember-simple-auth-oidc";

export default class ApolloService {
  @service session;

  link(...args) {
    const httpLink = super.link(...args);

    const afterware = onError(error => {
      const { networkError } = error;

      if (networkError.statusCode === 401) {
        handleUnauthorized(this.session);
      }
    });

    return afterware.concat(httpLink);
  }
}
```

[Ember Simple Auth v4.1.0](https://github.com/simplabs/ember-simple-auth/releases/tag/4.1.0) 
encourages the manual setup of the session service in the `beforeModel` of the 
application route.

```js
// app/routes/application.js

import Route from "@ember/routing/route";
import { inject as service } from "@ember/service";

export default class ApplicationRoute extends Route {
  @service session;

  async beforeModel() {
    await this.session.setup();
  }
}
```

### Logout / Explicit invalidation

There are two ways to invalidate (logout) the current session:

```js
session.invalidate();
```

The session `invalidate` method ends the current ember-simple-auth session and therefore performs a
logout on the ember application. Note that the session on the authorization server is not invalidated
this way and a new token/session can simply be obtained when doing the authentication process again.

```js
session.singleLogout();
```

The session `singleLogout` method will invalidate the current ember-simple-auth session and after that
call the `end-session` endpoint of the authorization server. This will result in a logout of the
ember application and additionally invalidate the session on the authorization server which will logout
the user of all applications using this authorization server!

## Configuration

The addon can be configured in the project's `environment.js` file with the key `ember-simple-auth-oidc`.

A minimal configuration includes the following options:

```js
// config/environment.js

module.exports = function (environment) {
  let ENV = {
    // ...
    "ember-simple-auth-oidc": {
      host: "http://authorization.server/openid",
      clientId: "test",
      authEndpoint: "/authorize",
      tokenEndpoint: "/token",
      userinfoEndpoint: "/userinfo",
    },
    // ...
  };
  return ENV;
};
```

Here is a complete list of all possible config options:

**host** `<String>`  
A relative or absolute URI of the authorization server.

**clientId** `<String>`  
The oidc client identifier valid at the authorization server.

**authEndpoint** `<String>`  
Authorization endpoint at the authorization server. This can be a path which
will be appended to `host` or an absolute URL.

**tokenEndpoint** `<String>`  
Token endpoint at the authorization server. This can be a path which will be
appended to `host` or an absolute URL.

**endSessionEndpoint** `<String>` (optional)  
End session endpoint endpoint at the authorization server. This can be a path
which will be appended to `host` or an absolute URL.

**userinfoEndpoint** `<String>`  
Userinfo endpoint endpoint at the authorization server. This can be a path
which will be appended to `host` or an absolute URL.

**afterLogoutUri** `<String>` (optional)  
A relative or absolute URI to which will be redirected after logout / end session.

**scope** `<String>` (optional)  
The oidc scope value. Default is `"openid"`.

**expiresIn** `<Number>` (optional)  
Milliseconds after which the token expires. This is only a fallback value if the authorization server does not return a `expires_in` value. Default is `3600000` (1h).

**refreshLeeway** `<Number>` (optional)  
Milliseconds before expire time at which the token is refreshed. Default is `30000` (30s).

**tokenPropertyName** `<String>` (optional)  
Name of the property which holds the token in a successful authenticate request. Default is `"access_token"`.

**authHeaderName** `<String>` (optional)  
Name of the authentication header holding the token used in requests. Default is `"Authorization"`.

**authPrefix** `<String>` (optional)  
Prefix of the authentication token. Default is `"Bearer"`.

**loginHintName** `<String>` (optional)  
Name of the `login_hint` query paramter which is being forwarded to the authorization server if it is present. This option allows overriding the default name `login_hint`.

**amountOfRetries** `<Number>` (optional)  
Amount of retries should be made if the request to fetch a new token fails. Default is `3`.

**retryTimeout** `<Number>` (optional)  
Timeout in milliseconds between each retry if a token refresh should fail. Default is `3000`.

## Contributing

### Installation

- `git clone git@github.com:adfinis-sygroup/ember-simple-auth-oidc.git`
- `cd ember-simple-auth-oidc`
- `yarn install`

### Linting

- `yarn lint` – Runs all linting tasks

### Running tests

- `yarn test` – Runs all linting and test tasks
- `yarn test:ember` – Runs the test suite on the current Ember version
- `yarn test:ember --server` – Runs the test suite in "watch mode"
- `yarn test:ember-compatibility` – Runs the test suite against multiple Ember versions

### Running the dummy application

- `yarn start`
- Visit the dummy application at [http://localhost:4200](http://localhost:4200).

For more information on using ember-cli, visit [https://ember-cli.com/](https://ember-cli.com/).

## License

This project is licensed under the [LGPL-3.0-or-later license](LICENSE).
