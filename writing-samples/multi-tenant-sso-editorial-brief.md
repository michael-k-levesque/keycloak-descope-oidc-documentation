# Editorial and SEO Brief — Multi-Tenant SSO for B2B SaaS

**Author:** Michael K. Levesque  
**Publication:** Independent technical-writing portfolio sample  
**Article:** [Multi-Tenant SSO for B2B SaaS: Secure Tenant Discovery, IdP Routing, and Account Linking](multi-tenant-sso-b2b-saas.md)

## Editorial objective

Explain the difference between discovering a customer's SSO connection, authenticating a user, resolving an application identity, and authorizing access to tenant-owned resources. The intended outcome is a clear decision guide for developers and security engineers designing B2B SaaS authentication—not a claim that the example architectures have been implemented or security-certified.

## Intended reader

Software engineers implementing enterprise SSO, B2B SaaS architects reviewing tenant and identity models, and application security practitioners checking cross-tenant trust boundaries. Readers should know basic web authentication and API concepts; the article defines the distinctions most likely to cause authorization mistakes.

## Search intent and phrase hypotheses

**Primary phrase hypothesis:** multi-tenant SSO for B2B SaaS.

**Supporting topic phrases:** IdP routing; home realm discovery; SSO tenant discovery; SAML and OIDC; account linking; tenant authorization; Descope multi-tenant SSO.

These are **editorial hypotheses**, not measurements. No keyword-volume, ranking, traffic, conversion, or competitor-performance data is claimed.

## Reader questions addressed

- How does a B2B SaaS application select the intended customer's IdP?
- What changes when a domain is shared or a tenant has more than one SSO connection?
- Why does successful SSO authentication not establish authorization to a tenant's invoices or other resources?
- Why is email-only account linking risky, and which Descope settings merit verification?
- Which negative-path checks catch routing and tenant-isolation mistakes before rollout?

## Article architecture and distinctive value

The article begins with an at-a-glance four-decision comparison, then presents routing options, a tenant-scoped invoice API example, a five-stage authorization design sequence, a concise account-linking section, and five rollout checks. Its principal technical distinction is **tenant association is not the same as resource authorization**, including where domain rules automatically associate users with tenants.

The Node.js session-validation passage also highlights that Descope's basic `validateSession` example does not enforce an intended audience unless the `audience` option is supplied. It does not present this conceptual passage as runnable, environment-validated sample code.

## Sources and editorial safeguards

Product behavior is anchored to current [Descope SSO documentation](https://docs.descope.com/auth-methods/sso), [project-level SSO settings](https://docs.descope.com/auth-methods/sso/settings), [tenant configuration](https://docs.descope.com/management/tenant-management/tenant), [SSO account-merging risks](https://docs.descope.com/sso/merging-sso-identities-risk), [multiple SSO providers](https://docs.descope.com/sso/multi-sso), and [backend validation](https://docs.descope.com/sessions/validation/backend). Protocol/security principles draw on [OpenID Connect Core §5.7](https://openid.net/specs/openid-connect-core-1_0.html#ClaimStability), [RFC 8725](https://www.rfc-editor.org/rfc/rfc8725), and [OWASP multi-tenant guidance](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html).

Descope's settings and merging-risk pages describe default account conversion differently. The article flags that discrepancy and advises checking the deployed configuration instead of asserting an unverified universal default.

## Independent evidence and limits

The accompanying [Keycloak–Descope case study](../README.md) documents one successful nonproduction OIDC sign-in, Descope session response, and JIT-associated user. It does **not** validate the additional multi-tenant routing, identity-conversion, enforcement, workspace-switching, or API-isolation mechanisms. The article distinguishes documentation-backed design guidance from this bounded observation.

This writing sample was independently developed. It was not commissioned, reviewed, or endorsed by Descope or Pepper. This brief records editorial decisions and intended audience; it is not an official client brief or product approval.