# Security, Privacy, and Release Controls

## Laboratory safeguards and their limits

The experiment used a separate Keycloak realm and confidential OIDC client, a dedicated Descope tenant, a restricted Nginx proxy, and a temporary HTTPS tunnel. This reduced accidental coupling with the existing Grafana `portfolio-lab` integration. It is **not evidence of a production security assessment** or a guaranteed safe internet-facing Keycloak deployment.

In particular, HTTPS to ngrok does not by itself establish end-to-end transport protection from Nginx to Keycloak, hardened headers, network segmentation, rate limits, account recovery policy, monitoring, key rotation, or protective controls on the admin UI.

## Private artifacts (excluded from this GitHub candidate)

- Keycloak database archive, checksum, and its H2 internal contents
- Nginx's actual runtime `default.conf` and host-specific filesystem paths
- IdP secret, Descope management keys, passwords, auth codes, session cookies, refresh tokens, JWTs
- Unredacted authentication screenshots showing tokens or other sensitive values
- Original Descope Flow JSON pending distribution/third-party license check
- Full Docker runtime dumps and detailed personal environment metadata

The exact temporary ngrok hostname is omitted from the public docs even though its previous existence was visible in development screenshots. Don't repurpose that hostname as a durable setup URL.

## GitHub publication gate

- [ ] Review these statements and confirm author approval
- [ ] Review the exact Git repository contents **and full reachable history**, not just the current working tree, for secrets, personal data and old screenshots
- [ ] Verify screenshot rights and redact all captured session/token material before adding images
- [ ] Confirm current Descope/Keycloak configuration syntax and versions if converting the case study into a runnable tutorial
- [ ] Confirm whether redistributing exported Flow JSON is permitted; this candidate intentionally omits it
- [ ] Validate Mermaid diagrams render on GitHub; confirm Markdown links resolve
- [ ] Obtain final technical review of material claim/evidence relationships
- [ ] Avoid suggesting a production-grade deployment or an official affiliation with Pepper or Descope

## Production concerns (out of scope)

A production rollout would need a supported external database; durable, controlled HTTPS ingress and upstream transport; secret storage/rotation; dedicated network access controls; admin endpoint protection; user lifecycle and MFA/recovery design; session revocation and logout semantics; application-side JWT validation with expected issuer/audience; tenant isolation/RBAC tests; monitoring; documented maintenance and recoverability; and independent threat modeling. No compliance or AAL certification is claimed.

## Recovery status

An October 18, 2026 Keycloak archive was confirmed to contain an H2 database and to pass SHA-256 verification; Nginx configuration was copied with restrictive file permissions. Restoration was **not** attempted. Evidence preservation does not equal verified disaster recovery.
