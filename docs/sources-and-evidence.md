# Sources and Evidence Register

**Assessment date:** 2026-10-18. This case study draws on (a) primary protocol/product documentation and (b) directly observed lab evidence. An observed console output must not be mislabeled as vendor policy.

## Technical references

| Ref | Primary source | What it supports |
|---|---|---|
| R-01 | [OpenID Connect Core 1.0, §3.1 Authorization Code Flow](https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowAuth) | Separation of authorization redirect from confidential-client code exchange; ID token as authentication assertion |
| R-02 | [OpenID Connect Discovery 1.0](https://openid.net/specs/openid-connect-discovery-1_0.html) | Discovery document and issuer/endpoint consistency concepts |
| R-03 | [Descope: Single Sign On with Flows](https://docs.descope.com/auth-methods/sso/with-flows) | An SSO action can be included in a Descope Flow and use OIDC or SAML |
| R-04 | [Descope: JIT Provisioning](https://docs.descope.com/sso/jit-provisioning) | On first SSO login, a configured tenant connection can create a user and update mapped fields later |
| R-05 | [Descope: SSO overview](https://docs.descope.com/auth-methods/sso) | Descope supports OIDC- and SAML-based federation |
| R-06 | [Keycloak: Server Administration Guide](https://www.keycloak.org/docs/latest/server_admin/index.html) | Client administration and realm OIDC endpoint layout |

## Direct observational evidence (private source custody)

| ID | Source type | Supported observation | Limits |
|---|---|---|---|
| E-01 | Descope tenant settings screenshots | OIDC client ID, Authorization Code, scopes, public realm endpoints, JIT option | Does not expose/verify secret equality |
| E-02 | OIDC metadata curl output | Public HTTPS `issuer` responded | Does not verify every OIDC endpoint end-to-end |
| E-03 | Descope Flow Builder + exported JSON | Four internal task nodes; saved minimal Flow | Does not include tenant IdP configuration |
| E-04 | Terminal outputs + Keycloak event logs | HTTP 401 response-path check; explicit `CODE_TO_TOKEN_ERROR`; client ID found for false secrets | Deliberate failures are not successful exchanges |
| E-05 | Browser/Runner screenshots | Sign-in page reached; later `Authentication Completed!` and Descope session response | Screenshots may contain sensitive token material, excluded |
| E-06 | Descope Users/user-details screenshot | New active OIDC user and tenant-aware login ID | Does not test lifecycle or authorization |
| E-07 | Ubuntu backup terminal output | Keycloak archive, H2 file inclusion, SHA-256 OK; Nginx backup copied | Does not establish successful restoration |

Evidence images, exported originals, and backup archives are stored by the lab operator, **not** redistributed in this candidate. No claimed successful test in this table is based solely on a draft diagram.

## Outstanding source gaps before a fully reproducible setup tutorial

The exact installed Keycloak build and Docker image tags, confidential-client redirect URI, actual proxy rule allowlist, future-stable public hostname, and clean-room deployment/negative-test procedure are not documented by currently available nonsecret exports. Do not invent these details in the published case study.
