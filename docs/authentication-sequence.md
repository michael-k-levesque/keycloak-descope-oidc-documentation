# OIDC Authentication Sequence

OIDC's Authorization Code Flow separates a **browser/front-channel authorization step** from a **client/back-channel token request**. The user authenticates at Keycloak; Descope, acting as the OIDC client/relying party in this lab, handles the code exchange and provides the application's session. The [OpenID Connect Core specification, §3.1](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) defines the protocol model.

```mermaid
sequenceDiagram
  autonumber
  actor U as Test user
  participant B as Browser
  participant D as Descope Flow / OIDC handler
  participant T as ngrok HTTPS + Nginx proxy
  participant K as Keycloak (descope-lab)
  U->>B: Start test Flow with tenant context
  B->>D: Start SSO action
  D-->>B: Redirect to IdP authorization URL
  B->>T: GET Keycloak authorization/login path
  T->>K: Proxy request to Keycloak
  K-->>T: Login response
  T-->>B: Keycloak sign-in page
  U->>B: Submit realm test-user credentials
  B->>T: Submit login via public IdP origin
  T->>K: Proxy authentication request
  K-->>B: Authorization response / browser redirect
  B->>D: Return authorization code via registered callback
  D->>T: Back-channel token request + client authentication
  T->>K: Forward token request
  K-->>D: Token response via reverse path
  D->>D: Process IdP result; create user/session
  D-->>B: Flow completion and Descope session response
```

**Fig. 2:** Logical OIDC authorization-code exchange based on the observed sign-in path. The browser return, server-to-server exchange, and success result were evidenced by the UI and Keycloak events; message-level wire packets were not independently captured. The diagram is a protocol-level explanation, not evidence of each cryptographic validation step.

## Trust boundaries

1. **Browser ↔ Descope:** User starts the authentication Flow, receives an IdP redirect, and later receives a Descope session result. Merely viewing JWT claims is not signature verification.
2. **Browser ↔ Keycloak through tunnel:** Browser navigates to a temporary HTTPS domain, but Keycloak ultimately handles credentials. The tunnel/proxy must not expose unrelated admin services.
3. **Descope ↔ Keycloak token endpoint:** Server-to-server exchange authenticates the OIDC client. Client secret material stays out of browser and source control.
4. **Federated identity → Descope user:** A Keycloak authenticated subject was mapped to the tenant-associated Descope login; a new active OIDC user appeared after sign-in.
5. **Authentication ≠ authorization:** The completed sign-in alone does not validate application resource access, role mapping or tenant-enforcement rules.

## Why the `issuer` mattered

OIDC discovery provides a public `issuer`. Its value must be consistently represented in provider metadata and the tokens being validated. A separate Grafana/Keycloak experiment had revealed mismatched internal and browser-visible issuer expectations; for this Descope realm, the team verified a public HTTPS issuer in discovery. See [OIDC discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) and [Keycloak's documented OIDC endpoints](https://www.keycloak.org/docs/latest/server_admin/index.html).
