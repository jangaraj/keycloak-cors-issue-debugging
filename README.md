# Problem

CORS issue in the browser, when app is trying to reach Keycloak

> Cross-Origin Request Blocked

# Recommendations

These recommendations are based on real experience. I'm sure there can be also another success path for CORS issues, but you will need more deep knowledge (see recommended doc section), so I would recommend to stick with these basic rules:

1. Configure `Web Origins` of used OIDC client in the Keycloak correctly

   That's `Origin` (first part of "URL"), which is calling Keycloak - usually "URL" of some SPA (Vue, React, Angular, JS) application/web site.
   What is `Origin`? - https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin
   > Origin: \<scheme\> "://" \<hostname\> \[ ":" \<port\> \]
  
   So `https://domain.com/*` is not a valid origin (it should be only `https://domain.com`).
   Examples of valid origins `https://1.1.1.1:8443,https://domain.com,http://domain.com:8888`.
   Examples of invalid origins `https://1.1.1.1:8443/,https://domain.com/path,http://domain.com:8888/*`.
  
2. Don't use any magic "regexp", e.g. `*,+` in the Origin OIDC Keycloak client configuration, be explicit

   https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Access-Control-Allow-Headers
  
   > In requests with credentials, it is treated as the literal header name "*" without special semantics. Note that the Authorization header can't be wildcarded and always needs to be listed explicitly.
 
3. Full Scope Allowed can't be enabled

   Make sure `Full Scope Allowed` is not `ON` in the Keycloak client `Scope Mapping` configuration
 
4. Don't use `localhost` for the development

   It is not saying not using local machine, just don't use `localhost` in the browser. Local machine has also local IP of your network, e.g. `192.168.0.133` , so I would use it everywhere (in the browser, in the OIDC client Origin configuration, ....). All SPA dev servers can bind also your local IP (e.g. use binding interace `0.0.0.0`), so they can run also on local IP, not just on `localhost` or `127.0.0.1` interface. You can use local hosts file and develop app on fake local "domain" directly. Problem is that some browser may have very likely own config for `localhost` - https://stackoverflow.com/questions/10883211/deadly-cors-when-http-localhost-is-the-origin

5. Use browser network console to inspect prefligh request

   [Preflight request](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request) (`OPTIONS` method) is saying to Keycloak "what" is requested by the app and then Keycloak returns "what" is allowed actually. Simple `curl` debugging on Linux of preflight request - app is developed on `http://192.168.0.133:8080`, so that's used as an origin:
   ```
   $ curl -X OPTIONS \
     -H "Origin: http://192.168.0.133:8080" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: authorization,x-requested-with" \
     -k https://<keycloak-host>/auth/realms/<realm>/protocol/openid-connect/token \
     --silent --verbose 2>&1 | grep Access-Control
   > Access-Control-Request-Method: POST
   > Access-Control-Request-Headers: authorization,x-requested-with
   < Access-Control-Allow-Headers: Origin, Accept, X-Requested-With, Content-Type, Access-Control-Request-Method, Access-Control-Request-Headers, Authorization
   < Access-Control-Allow-Origin: http://192.168.0.133:8080
   < Access-Control-Allow-Credentials: true
   < Access-Control-Allow-Methods: POST, OPTIONS
   < Access-Control-Max-Age: 3600
   ```

   Requested details from pre preflight request e.g. `Origin, Access-Control-Request-Method, Access-Control-Request-Headers` 
   must be included in the preflight response e.g. `Access-Control-Allow-Origin, Access-Control-Request-Method, Access-Control-Request-Headers`. If they are not or they are not matching, then browser will denies to execute request. This is example of preflight request for token endpoint, but each Keycloak URL may have own preflight request, which should be inspected.
  
6. Use correct OIDC flow

   So for SPA it is `Authorization Code Flow with Proof Key for Code Exchange (PKCE)` usually (so used OIDC client must be public). I can imagine also `Client Credentials Flow` or `Direct Access Grant Flow` can be used. `Implicit Flow` is [deprecated flow](https://developer.okta.com/blog/2019/05/01/is-the-oauth-implicit-flow-dead), so don't use it anymore.
   I would recommend to read https://developer.okta.com/docs/concepts/oauth-openid/#what-kind-of-client-are-you-building
   
   CORS issue to auth endpoint e.g. `https://<keycloak-host>/auth/realms/<realm>/protocol/openid-connect/auth` indicates wrong backend/API implementation or used flow. XHR request shouldn't never reach auth Keycloak endpoint (that is designated only for full user browser access and not XHR access)
  
7. Use certified OIDC library

   Unless, you are using simple flows - but mature library may make your life easy also in their case.
   See: https://openid.net/developers/certified/ OIDC is a standard SSO standard. You really don't need to use any library with `keycloak` in the name. But used library must support OIDC SSO protocol. Actually, I would avoid any `keycloak` libraries completely. It may introduces vendor dependency and you will need to change the code in the future, when you will decide to change used Identity provider. Humble recommendation for Angular: https://github.com/damienbod/angular-auth-oidc-client

# Example implementations

- React: https://github.com/dasniko/keycloak-reactjs-demo
- Angular: https://github.com/damienbod/angular-auth-oidc-client/tree/main/projects

# Recommended doc

- https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/Errors

# Please don't contact me

These recommendations may help you. If not please don't contact me with the request for the help (unless you demand commercial paid support). Unfortunately, I don't have a capacity for that. Use another resources, e.g. https://keycloak.discourse.group/ and always mention how did you configure client and how did you debug the issue (preflight request can be very usefull to see what is requested and what is returned by Keycloak). Requests `I have CORS issue, so what to do` are useless. Provides details and then you will increase your chance that someone gives you some clue. Example of usefull problem report (it's clear which Keycloak version is used, how is OIDC client configured, how SPA auth code looks like, which library is used, what is the CORS issue):

1.) I use Keycloak 12.0.3 and OIDC client with this configuration (exported json model):

```
{
    "clientId": "vue",
    "description": "My test OIDC client for my Vue SPA app",
    "surrogateAuthRequired": false,
    "enabled": true,
    "alwaysDisplayInConsole": false,
    "clientAuthenticatorType": "client-secret",
    "redirectUris": [
        "https://192.168.11.133/*"
    ],
    "webOrigins": [
        "https://192.168.11.133/*"
    ],
    "notBefore": 0,
    "bearerOnly": false,
    "consentRequired": false,
    "standardFlowEnabled": true,
    "implicitFlowEnabled": false,
    "directAccessGrantsEnabled": true,
    "serviceAccountsEnabled": false,
    "publicClient": false,
    "frontchannelLogout": false,
    "protocol": "openid-connect",
    "attributes": {
        "saml.assertion.signature": "false",
        "saml.multivalued.roles": "false",
        "saml.force.post.binding": "false",
        "saml.encrypt": "false",
        "backchannel.logout.revoke.offline.tokens": "false",
        "saml.server.signature": "false",
        "saml.server.signature.keyinfo.ext": "false",
        "exclude.session.state.from.auth.response": "false",
        "backchannel.logout.session.required": "true",
        "client_credentials.use_refresh_token": "false",
        "saml_force_name_id_format": "false",
        "saml.client.signature": "false",
        "tls.client.certificate.bound.access.tokens": "false",
        "saml.authnstatement": "false",
        "display.on.consent.screen": "false",
        "saml.onetimeuse.condition": "false"
    },
    "authenticationFlowBindingOverrides": {},
    "fullScopeAllowed": true,
    "nodeReRegistrationTimeout": -1,
    "defaultClientScopes": [
        "web-origins",
        "role_list",
        "profile",
        "roles",
        "email"
    ],
    "optionalClientScopes": [
        "address",
        "phone",
        "offline_access",
        "microprofile-jwt"
    ],
    "access": {
        "view": true,
        "configure": true,
        "manage": true
    }
}
```

2.) Snippet of my authentication Vue code - I use `keycloak-js` library version 9.0.0:

```
const initOptions = {
  url: process.env.VUE_APP_KEYCLOAK_OPTIONS_URL,
  realm: process.env.VUE_APP_KEYCLOAK_OPTIONS_REALM,
  clientId: process.env.VUE_APP_KEYCLOAK_OPTIONS_CLIENTID,
  onLoad: process.env.VUE_APP_KEYCLOAK_OPTIONS_ONLOAD,
}
const keycloak = Keycloak(initOptions)

keycloak.init(
  { onLoad: initOptions.onLoad, promiseType: 'native', checkLoginIframe: false }
).then(async authenticated => {
  if (!authenticated) {
    window.location.reload()
    return
  } else {
    Vue.prototype.$keycloak = keycloak
    await store.dispatch('user/keycloakLogin', keycloak.token)
  }

  setInterval(() => {
    keycloak.updateToken(70).then((refreshed) => {
      if (refreshed) {
        console.log('Token refreshed')
        setToken(keycloak.token)
      }
    }).catch(error => {
      store.commit("snackbar/setSnack", {message:'Your access token was not refreshed automatically. Try full page reload in your browser. Error: ' + error, show: true});
      console.log('Failed to refresh token', error)
    })
  }, 60000)
      
}).catch(error => {
  console.log('Authentication failed: ', error)
})
```

Env variables:
```
VUE_APP_KEYCLOAK_OPTIONS_URL='https://192.169.11.133:8443/auth'
VUE_APP_KEYCLOAK_OPTIONS_REALM='master'
VUE_APP_KEYCLOAK_OPTIONS_CLIENTID='vue'
VUE_APP_KEYCLOAK_OPTIONS_ONLOAD='login-required'
```

3.) Recorded requests of the app with the Keycloak with recoded CORS issue (preflight request included) is available in the har file.

<attach har file - see [how to generate har file](https://www.inflectra.com/support/knowledgebase/kb254.aspx)>

You can use this report as an exercise and find obvious problem(s), which violate recommendations above.
