# Verification Matrix

**Method:** Actual user-executed console interactions, browser observation, terminal output and Keycloak event logs from October 18, 2026. “PASS” means the explicitly bounded observation occurred—not that the surrounding system is production-ready.

| Test ID | Test / expected evidence | Observed result | Verdict |
|---|---|---|---|
| V-01 | Public OIDC discovery emits realm issuer matching temporary HTTPS origin | Issuer inspected in discovery response | PASS |
| V-02 | Browser redirected from Descope SSO action to correct Keycloak realm | Keycloak `DESCOPE-LAB` sign-in page via ngrok | PASS |
| V-03 | Proxy forwards incomplete token POSTs to service | Local + public endpoints returned JSON HTTP `401` | PASS: reachability only |
| V-04 | Keycloak recognizes confidential client when supplied bad Basic credentials | `clientId="descope-sso"`, `invalid_client_credentials` | PASS: expected rejection |
| V-05 | Keycloak recognizes client ID with bad form credentials | Same event with form-based client ID | PASS: expected rejection |
| V-06 | Correct test-user sign-in completes via Flow Runner | “Authentication Completed!” and session response | PASS |
| V-07 | User record exists after first SSO sign-in | User shown Active, SSO OIDC; tenant-associated Login ID | PASS |
| V-08 | Exported Flow graph internally consistent | Four task nodes (`start`, `1`, `1.end`, `0`), edges resolvable | PASS: static JSON |
| V-09 | Keycloak archive readable and checksum matches | H2 `.mv.db` listed; SHA-256 verification output `OK` | PASS: archive only |
| V-10 | Isolated restore works | No isolated restore completed | NOT TESTED |
| V-11 | Application RBAC/tenant isolation prevents wrong-tenant access | No resource server or negative-case policy exercise | NOT TESTED |
| V-12 | Token refresh, logout propagation, session revocation | No end-to-end tests | NOT TESTED |
| V-13 | Browser compatibility/identity lifecycle under load | No multi-browser or load testing | NOT TESTED |
| V-14 | Console SSO warning accurately characterizes API/SDK starts | Flow SSO worked; separate API start was Cloudflare-blocked | INCONCLUSIVE |

## What would make the verification reproducible?

A future maintenance/test cycle should capture a sanitized, versioned Keycloak client configuration; the allowed reverse-proxy paths; exact redirect URI and flows; a controlled new browser session; and positive and negative tests. Execute these only after a safe private environment is ready. Don't convert planned steps into PASS results.

## Evidence custody

A private original Flow JSON is retained offline. Screenshots exist from the user-driven test but some show session information and must **not** be included in a public repository without redaction and review. The published table contains synthesized observations, not raw secrets.
