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

## Publication review (2026-10-19)

This is a **technical case study**, not an installable product or an independently audited security reference. The private GitHub repository was reviewed before changing its visibility.

| Review item | Result / boundary |
|---|---|
| Repository file inventory | 12 files on `main` verified |
| History review | Six commits reviewed; 15 unique text/blob revisions scanned for targeted credential, JWT, temporary tunnel hostname, and personal-path patterns; no matches found |
| Screenshots and raw sessions | No screenshots, tokens, captured traffic, or user-profile exports are included |
| Original Descope Flow JSON | Intentionally withheld; no redistribution permission assumed |
| Architecture figure | Approved vector SVG visually checked in GitHub; editable Mermaid source retained |
| Supporting Mermaid diagrams | Source and corresponding Markdown blocks reconciled; manual rendered-layout confirmation remains a presentation check |
| Source and claim boundaries | Observations distinguished from protocol guidance, unverified recovery, and production hardening |
| Public visibility | Requires repository owner to choose **Public** in GitHub settings |

A targeted text scan is not a guarantee against every secret or privacy issue; review future commits and additions independently. Final publication approval rests with the author.

### Additional work for a reproducible deployment guide

If this case study is expanded into a copy-and-run tutorial, first capture supported Keycloak/Descope versions, exact confidential-client redirect settings, reviewed proxy allowlist, secret-management procedure, fresh-environment setup and negative-path tests. **These are not claimed as completed in the present case study.** Do not add raw test screenshots without confirming rights and removing all session/token material.

## Production concerns (out of scope)

A production rollout would need a supported external database; durable, controlled HTTPS ingress and upstream transport; secret storage/rotation; dedicated network access controls; admin endpoint protection; user lifecycle and MFA/recovery design; session revocation and logout semantics; application-side JWT validation with expected issuer/audience; tenant isolation/RBAC tests; monitoring; documented maintenance and recoverability; and independent threat modeling. No compliance or AAL certification is claimed.

## Recovery status

An October 18, 2026 Keycloak archive was confirmed to contain an H2 database and to pass SHA-256 verification; Nginx configuration was copied with restrictive file permissions. Restoration was **not** attempted. Evidence preservation does not equal verified disaster recovery.
