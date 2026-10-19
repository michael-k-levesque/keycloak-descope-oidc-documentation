# Configuration Record and Boundaries

**Date of successful test:** 2026-10-18. **Scope:** Isolated Descope/Keycloak OIDC test, not the separate Grafana `portfolio-lab` integration.

## Component inventory

| Component | Confirmed from user-operated lab | Not independently established by this record |
|---|---|---|
| Docker host | Ubuntu under WSL 2; `keycloak-lab`, `grafana-lab`, `descope-oidc-proxy` names observed | Complete compose source, resource hardening, persistence of every container |
| Keycloak | Realm `descope-lab`; OIDC confidential client `descope-sso`; Standard Flow on; Client ID and Secret authenticator | Exact installed Keycloak version, production TLS configuration, full client policy export |
| Proxy | Nginx on private local port `8180` observed; token and discovery checks responded | Complete audited route allowlist and detailed reverse-proxy headers |
| Tunnel | HTTPS development forwarding via ngrok to local proxy | Permanent domain, production ingress, persistent availability |
| Descope | Free Plan, Non-Production project; independent tenant `keycloak-oidc-sso-lab`; OIDC scopes `openid profile email`; Authorization Code; Client Secret | Guaranteed feature availability for other plans, exact internal validation methods |
| Flow | `keycloak-oidc-sso-test-v1`, Interactive; START → SSO → END; exported JSON source reviewed | Flows as complete source for tenant SSO configuration |
| Test user | Dedicated Keycloak identity `descope.tester` was created with a non-personal test address | Broader provisioning lifecycle, SCIM mappings, organization RBAC |

## Saved Descope OIDC provider settings (non-secret subset)

- Provider name: `Keycloak`
- Client ID: `descope-sso`
- Client authentication: Client Secret; stored value confirmed masked, content **not** disclosed
- Grant type: Authorization code
- Scopes: `openid`, `profile`, `email`
- Discovery URL: `https://<temporary-idp-domain>/realms/descope-lab/.well-known/openid-configuration`
- Issuer: `https://<temporary-idp-domain>/realms/descope-lab`
- Authorization and token endpoints: populated from the same realm's published OIDC discovery document
- JIT provisioning: checked in the setup UI; subsequent Descope user creation observed
- Map IdP `sub` to Descope Login ID; mapping observed as tenant-aware suffix in user details
- Test Flow Runner input key: `tenant` → `keycloak-oidc-sso-lab`

The Keycloak client's exact **valid redirect URI** should be checked against its actual saved settings before claiming a reproducible configuration guide; it was not exported for this document. Do not assume the general Descope OAuth callback example is the definitive one in this specific tenant setup.

## What not to reproduce from this guide

The development configuration is intentionally **not** a public, copy-and-paste deployment recipe. Never expose the Keycloak administration console or unrelated realm routes through a general-purpose tunnel. A public OIDC discovery and token endpoint alone do not prove a proxy policy is hardened. Use current vendor documentation and a separately reviewed threat model for any real deployment.

## Backup and restore boundary

An offline Keycloak H2 data-volume archive was made after the successful SSO test; archive inclusion, `tar` readability and SHA-256 comparison passed. Keycloak was stopped during the archive and restarted. Nginx's configuration file was separately copied with restrictive permissions. **No restore test was performed**, and the backup is not represented as recoverable production infrastructure.
