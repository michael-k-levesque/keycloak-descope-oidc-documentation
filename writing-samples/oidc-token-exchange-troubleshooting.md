# OIDC Token Exchange Failures: A Layer-by-Layer Troubleshooting Guide for B2B SSO

By Michael K. Levesque

A user enters the correct credentials at an identity provider (IdP). The login page accepts them. The browser follows the redirect back to the relying party’s callback. Then, instead of a session, the application reports **“Token exchange with SSO provider failed.”**

The visible login succeeded, but OIDC Authorization Code Flow still requires a back-channel code exchange, identity validation, and application-session creation. **Identify the failed boundary before changing configuration.**

This field guide explains how, using a Keycloak–Descope lab that encountered an initial token-exchange error before completing SSO successfully.

## At a glance: where to look first

When no session appears, inspect tenant routing, token-request events, client authentication, redirects, discovery metadata, proxy behavior, and provisioning.

**Key principle:** A successful login page, an HTTP `401` from a token endpoint, and an application session are different kinds of evidence. None proves the others.

## Why a successful IdP login can still fail

OIDC adds authentication to OAuth 2.0. Its Authorization Code Flow separates browser navigation from server-side token exchange. Confidential clients also authenticate themselves during redemption. An ID token conveys identity; an access token is not interchangeable with it.

A B2B sign-in typically follows these steps:

1. **Select the tenant and SSO connection.** The application determines which organization's IdP should handle sign-in.
2. **Send the browser to the authorization endpoint.** The request includes values such as `client_id`, `redirect_uri`, `response_type=code`, and an OIDC `scope` containing `openid`. Use `state` to correlate the response and defend against request-forgery attacks, and PKCE where applicable; current OAuth security guidance recommends PKCE broadly.
3. **Authenticate the user at the IdP.** The browser interacts with the IdP, not with the application's token endpoint.
4. **Return an authorization code.** The browser follows a redirect to the registered callback. Receiving a code does not mean the code has been redeemed.
5. **Exchange the code on the back channel.** The relying party sends the code and required parameters to the IdP's token endpoint, over TLS. A confidential client also proves its registered client identity.
6. **Validate identity and establish a session.** After a successful token response and appropriate validation, the application or authentication platform resolves the user and creates its own session. Optional Just-in-Time (JIT) provisioning can create a first-time user record when configured; authentication-only SSO actions or connections deliberately avoid creating a Descope session or user.

Between steps 4 and 5, a successful browser journey can still turn into a failed back-channel exchange. Our [independent Keycloak–Descope lab](https://github.com/michael-k-levesque/keycloak-descope-oidc-documentation) used a temporary HTTPS ingress and Nginx proxy to connect Descope to Keycloak. This is a development case study, not a deployment pattern.

## A symptom-to-layer troubleshooting matrix

Start with what you observed, rather than assuming a root cause from a broad UI error.

| What you see | Likely investigation layer | First evidence to collect |
|---|---|---|
| Wrong tenant, wrong IdP, or no authorization redirect | Tenant / SSO routing | Tenant identifier, selected connection, Flow input |
| Browser cannot open the IdP login page | Discovery, DNS, TLS, ingress | Authorization URL, browser/network error, proxy access logs |
| IdP rejects the username or account | IdP user authentication | Realm or directory, user status, IdP login event |
| Browser returns with `error` instead of `code` | Authorization response | Redacted callback parameters, IdP authorization event |
| Application reports token exchange failure | Back-channel token request | IdP token event, provider response code, request correlation |
| IdP says `invalid_client` or logs `client_not_found` | Client authentication / registration | Client ID recognition, allowed authentication method, proxy handling |
| IdP says `invalid_grant` | Code redemption | Code age/reuse, redirect URI, PKCE verifier, client binding |
| Exchange succeeds but no application session appears | Identity mapping / session | Validated token claims, tenant membership, JIT policy, app logs |

Treat these as hypotheses. Confirm the error’s origin; gateways may return HTML errors.

## Step 1: Confirm the tenant and connection before debugging OAuth

Even perfect IdP connectivity cannot fix a Flow using the wrong tenant connection. Record these identifiers separately:

- The application tenant or organization identifier.
- The OIDC client ID registered with the external IdP.
- The IdP realm or issuer.
- The authentication Flow ID or login route used by the application.

These identifiers are not interchangeable. Descope routes SSO through the selected tenant. The `tenant` Flow parameter selects that tenant before rendering; an ordinary SSO action creates or updates a user and session unless the connection is authentication-only. Record tenant, Flow ID, and connection before editing credentials.

**Descope-specific note:** `E062605` indicates an SSO-provider token-exchange failure, not its exact cause. `E062014` instead concerns disabled JIT provisioning. See [Descope SSO troubleshooting](https://docs.descope.com/other-troubleshooting/sso-troubleshooting).

## Step 2: Read discovery metadata, then compare actual endpoints

An OpenID Provider’s `/.well-known/openid-configuration` document advertises its issuer and OIDC endpoints.

Inspect a Keycloak realm’s non-secret metadata:

```bash
# Example only: substitute the public HTTPS base URL of a test IdP.
IDP_BASE_URL="https://idp.example.org"
REALM="example"

curl --fail-with-body --silent --show-error \
  "$IDP_BASE_URL/realms/$REALM/.well-known/openid-configuration" \
  | jq '{issuer, authorization_endpoint, token_endpoint, jwks_uri}'
```

The issuer must identify the intended provider. Check each advertised endpoint against the deployment’s actual front-channel and back-channel routing; the endpoints need not all use the same public hostname when a separate, explicitly configured back channel is supported. Reading metadata does **not** validate a token exchange.

Check four things:

1. **Issuer identity:** Does the configured issuer exactly match the issuer claimed by the IdP's metadata and expected during ID-token validation?
2. **Endpoint origin:** Are the authorization and token URLs appropriate for the browser and relying party respectively? An internal Docker hostname or stale tunnel address is wrong **if it is not reachable or intended from the system using that endpoint**.
3. **TLS and reachability:** Can the *relying party's environment*, not merely your laptop browser, reach the token endpoint and trust its certificate chain?
4. **Proxy-aware hostname configuration:** If Keycloak sits behind a reverse proxy, does its configured public hostname and trusted forwarded-header behavior produce the correct URLs?

Keycloak's current [hostname guidance](https://www.keycloak.org/server/hostname) documents fixed and dynamic URL strategies, including deliberately separate private back-channel addresses. A dynamically resolved hostname can be useful in controlled tests, but trusting unsanitized forwarded host headers creates security risks. Its [reverse-proxy guidance](https://www.keycloak.org/server/reverseproxy) covers trusted forwarding, restricted exposure, and TLS termination/re-encryption tradeoffs for different deployment models.

**Common trap:** A working discovery request from your laptop does not prove the cloud relying party can reach the token endpoint.

## Step 3: Distinguish the two redirect URL configurations

Federated B2B login may involve two different redirect relationships:

**OIDC redirect URI (registered at the IdP):** The external IdP is configured to return an authorization response to the OIDC relying party’s callback endpoint. The URI supplied in the authorization request must match an allowed registration value, and the token request must satisfy the protocol's redirect consistency rules.

**Application post-authentication redirect:** After the authentication platform completes SSO, it may send the browser back to the application. Platforms such as Descope can separately restrict these application URLs.

Fixing an application landing URL does not repair a mismatch in the OIDC redirect URI registered with the IdP, or vice versa.

Compare the exact values observed in the failing run: scheme (`https` versus `http`), hostname, port where relevant, path, and trailing slash. Avoid inferring the callback from a generic integration tutorial when the actual tenant configuration can be inspected.

A changed `redirect_uri` can cause `invalid_grant` at the token endpoint. Descope also documents application redirect-allowlisting errors. Identify the error's origin before editing either setting.

## Step 4: Identify the client before blaming the secret

A confidential client must authenticate to the token endpoint, but **client recognition** and **secret validity** are different checks. OAuth's `invalid_client` may mean unknown client, missing credentials, or an unsupported method. IdP events often narrow the diagnosis.

For Keycloak, inspect these client settings together:

- **Client ID:** Does it exactly match the value configured at the relying party?
- **Client authentication:** Is authentication enabled for the confidential client?
- **Authenticator and allowed method:** Does the server expect a client secret in HTTP Basic authentication, in the form body, or another supported method?
- **Grant/flow settings:** Is the Authorization Code (Keycloak Standard) Flow enabled?
- **Secret status:** Is the stored secret current, and has it been rotated since the integration was configured?

Keycloak can accept Basic authentication or, when allowed, body parameters. A correct secret will still fail if the client permits only Basic but the relying party sends credentials in the body.

**Security warning:** Never record production secrets, codes, cookies, or tokens in tickets or logs. Base64-encoded Basic credentials are not protected secrets. Use isolated test clients and redacted output.

With a disposable test client, an intentionally wrong secret may still generate a log identifying the expected client. That confirms recognition for that request—not a successful exchange or an exact reproduction of the relying party's traffic.

## Step 5: Interpret `invalid_grant` without guessing

After client authentication, `invalid_grant` may indicate an expired, reused, revoked, or mismatched code, a redirect discrepancy, or a grant issued to another client.

Use a fresh login per test: authorization codes are short-lived and normally single-use. Replaying a captured code can create a misleading new failure.

Investigate the following questions:

- **Timing:** Did the token exchange occur promptly after the callback, or did a retry queue or paused debugger delay it?
- **Reuse:** Did two application instances or a retry mechanism attempt to redeem the same code?
- **Client binding:** Was the code issued to the same client attempting redemption?
- **Redirect consistency:** Does the token request use the exact relevant redirect URI from the authorization request?
- **PKCE consistency:** If the request used Proof Key for Code Exchange (PKCE), did the token request include the correct `code_verifier` matching the original code challenge?

Under [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636), a PKCE verifier mismatch produces `invalid_grant`. Check whether PKCE was required and compare the actual parameters.

`unauthorized_client` instead means the client cannot use the requested grant type.

## Step 6: Check the back-channel route, not only the browser path

Federation takes two distinct network paths:

**Front channel:** Browser → public IdP authorization/login endpoint → browser callback.

**Back channel:** Relying party or authentication platform → token endpoint → response returned to the relying party.

A proxy may pass login traffic while blocking or misrouting the token request, altering forwarded-host behavior, or preventing cloud egress.

Capture only safe request metadata: method, path, upstream, status, timestamp, and correlation ID. Never log codes, credentials, cookies, or raw tokens.

For connectivity checks, begin with the public discovery endpoint and the proxy's known route allowlist. A deliberately incomplete token-endpoint `POST` may produce an OAuth error, but the exact status depends on server behavior. Seeing an HTTP `400` or `401`, **together with service-identifying response details or a correlated IdP event**, can provide evidence that a request reached the token-processing path; an HTTP status alone cannot. None of this proves a valid, authenticated authorization-code exchange.

An HTML gateway `403` is not an OAuth JSON error from the IdP. Inspect the response's origin and type before assigning fault.

A temporary tunnel is not production ingress. Protect routes, forwarding headers, upstream TLS, administration access, and the Keycloak management port.

## Step 7: Separate token success from session and user-provisioning success

Once the IdP returns tokens, another boundary remains: the relying party must validate the identity response and associate it with the correct application user and tenant. A B2B authentication platform may additionally create a first-time user record through JIT provisioning.

Check the following independently:

- Was a valid OIDC identity result processed by the relying party?
- Did the external subject (`sub`) map to the intended login identifier?
- Was the expected tenant identified consistently throughout the flow?
- Was JIT provisioning enabled, or was the user already expected to exist?
- Was the user created or updated successfully?
- Did the platform issue an application session and return the browser to its approved destination?

Descope's normal SSO Flow action is documented to create or update the user and issue a session, while its Authenticate Only variant does not. An **authentication-only SSO connection** can also change the result of an otherwise normal SSO action. If the user is missing after sign-in, confirm which action was used before interpreting the result as failed provisioning.

A missing theme asset may affect presentation without blocking authentication. A correct-looking screen is no substitute for session evidence.

**Authentication is not authorization.** Successful SSO says nothing conclusive about resource authorization, tenant isolation, or RBAC.

## Field example: Keycloak recognized no client during one exchange

An independent integration lab connected a Keycloak realm to a Descope tenant over OIDC Authorization Code Flow. The browser reached the correct Keycloak login, but the Descope Flow Runner initially reported a token-exchange failure.

A sanitized Keycloak event identified the failing phase:

```text
CODE_TO_TOKEN_ERROR realmName="descope-lab"
clientId="null" error="client_not_found"
grant_type="authorization_code"
```

Keycloak failed to resolve a client identity for **that request**. Its original headers and body were unavailable.

The operator ran controlled isolation tests:

1. Incomplete token requests to the local and public endpoints received JSON error responses, confirming a response path rather than a successful exchange.
2. Deliberately incorrect credentials sent with Basic authentication produced `invalid_client_credentials`, with Keycloak identifying the expected test client.
3. A corresponding form-body credential test also resulted in the client being identified and the fake secret rejected.
4. The saved Descope configuration showed the expected client ID, Authorization Code grant, and public realm endpoints. Its masked secret could not be compared directly from the screenshot.
5. The existing client secret was re-entered in Descope, the correct realm test user was used, and a subsequent interactive SSO run completed successfully.

A subsequent session response and newly active user independently confirmed one successful SSO and JIT provisioning test.

What caused the original failure? **The evidence does not prove a single cause.** Re-entering a secret before the successful retry does not establish that the old secret was incorrect. The original token POST was not captured in sanitized form; the client ID might have been absent, the authentication format might have differed, or another factor could have been involved.

The [public case study](https://github.com/michael-k-levesque/keycloak-descope-oidc-documentation) preserves this distinction between observed success and unproved root cause.

## A repeatable diagnostic workflow for engineering teams

When an SSO ticket arrives, change one setting at a time and follow an evidence-first process:

**1. Timestamp the failure.** Record the tenant, Flow or application ID, IdP realm, and a correlation ID. Use separate IDs for separate attempts. Collect no actual session token or authorization code in the ticket.

**2. Locate the last successful boundary.** Was there an IdP page? A successful user login? A callback containing a code? A token response? A completed application session?

**3. Read both sides.** Correlate the relying party's error with the IdP's authorization and code-to-token events. Do not diagnose from a generic UI message alone.

**4. Check high-confidence configuration.** Confirm discovery issuer, token endpoint, client ID, client-authentication method, grant settings, and exact callback URI before rotating secrets or changing proxy rules.

**5. Isolate the path safely.** Compare local and public endpoint responses with non-secret test requests; distinguish gateway rejection, service reachability, client recognition, and successful exchange.

**6. Change one variable.** Rerun a fresh authentication attempt, save the timestamped result, and record whether the hypothesis was confirmed, weakened, or still unresolved.

**7. Verify beyond the login screen.** Confirm session issuance, user association, tenant context, and relevant negative tests. Mark anything not tested as unverified.

Maintain an ownership register for endpoints, redirect URIs, credentials, tenant mappings, and rotation procedures to reduce cross-tenant mistakes.

## Frequently asked questions

### Does a successful Keycloak login mean OIDC SSO is working?

No. The user may authenticate successfully at Keycloak while the relying party fails to redeem the authorization code, validate the identity response, or create its application session. Confirm those phases independently.

### What does `invalid_client` mean at the token endpoint?

It means client authentication failed. According to [OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749#section-5.2), possibilities include an unknown client, absent credentials, or an unsupported authentication method. Check the IdP's more specific events before concluding the secret is wrong.

### Can a reverse proxy cause an OIDC code-exchange failure?

Yes. Incorrect endpoint routing, TLS trust problems, ingress restrictions, or mishandled forwarded headers can break the back channel even when the browser reaches the login page. Validate the relying party's outbound path and the proxy's safe routing logs.

### Is `invalid_grant` always an expired authorization code?

No. It can also involve a reused or invalid code, a redirect URI mismatch, a code bound to another client, or a PKCE verifier mismatch. The exact cause depends on the IdP's checks and the corresponding event evidence.

### Does JIT provisioning happen before the token exchange?

Not as a substitute for successful federation. In a typical SSO flow, the authentication result is processed first; configured JIT behavior can then create or update an application user. Descope also offers an Authenticate Only action and authentication-only SSO connections, neither of which should be treated as ordinary session-creating sign-ins.

## The bottom line

OIDC troubleshooting begins with: *Which system rejected which request, and what evidence proves it?* Check tenant routing, discovery, client and redirect settings, grants, and proxy behavior before confirming session results. Preserve uncertainty wherever the evidence stops.

---

### References and further reading

- [OpenID Connect Core 1.0 — Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth)
- [RFC 6749 — OAuth 2.0, Token Endpoint Error Response](https://datatracker.ietf.org/doc/html/rfc6749#section-5.2)
- [RFC 7636 — Proof Key for Code Exchange](https://datatracker.ietf.org/doc/html/rfc7636)
- [RFC 9700 — OAuth 2.0 Security Best Current Practice](https://www.rfc-editor.org/rfc/rfc9700.html)
- [Descope — SSO with Flows](https://docs.descope.com/auth-methods/sso/with-flows)
- [Descope — SSO Troubleshooting](https://docs.descope.com/other-troubleshooting/sso-troubleshooting)
- [Descope — SSO Authenticate Only](https://docs.descope.com/flows/actions/sso-authenticate-only)
- [Descope — Multiple SSO Providers and Authentication-Only Connections](https://docs.descope.com/sso/multi-sso)
- [Descope — How to Fix Redirect URI and Client Credentials Errors](https://www.descope.com/blog/post/redirect-uri-client-credentials)
- [Keycloak — Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/index.html)
- [Keycloak — Configuring the Hostname](https://www.keycloak.org/server/hostname)
- [Keycloak — Configuring a Reverse Proxy](https://www.keycloak.org/server/reverseproxy)
