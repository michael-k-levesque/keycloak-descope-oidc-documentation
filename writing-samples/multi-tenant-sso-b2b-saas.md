# Multi-Tenant SSO for B2B SaaS: Secure Tenant Discovery, IdP Routing, and Account Linking

By Michael K. Levesque

In a multi-tenant B2B application, knowing which identity provider (IdP) should authenticate a user is not the same as knowing which customer's resources that user may access. Depending on configuration, an email domain can help route sign-in or automatically associate a user with a tenant. Neither a routing match nor tenant membership alone establishes permission to access a protected API resource.

That distinction shapes a secure single sign-on (SSO) design. The application must identify the intended organization, select the appropriate SAML or OpenID Connect (OIDC) connection, establish the authenticated identity, and enforce permissions for the requested tenant and resource. A successful SSO callback is only part of that process.

This guide explains those boundaries, compares practical tenant-discovery approaches, and shows where Descope's documented B2B SSO capabilities fit.

## At a glance: four separate decisions

| Decision | What it determines | What does not establish it |
| --- | --- | --- |
| **Tenant discovery** | Which customer organization the sign-in attempt concerns | An email address as proof of membership |
| **IdP routing** | Which configured SAML or OIDC connection handles authentication | A generic assumption that every tenant uses one IdP |
| **Identity resolution** | Which principal was authenticated and how it maps to an application user | An unvalidated or coincidentally matching email string |
| **Authorization** | Whether the principal may access the requested tenant, operation, and resource | A valid login or a tenant ID supplied by the browser |

**The design rule:** Treat tenant selection as a routing input until the user has authenticated. After authentication, the application still needs trusted tenant context and a server-side authorization decision. OWASP's [Multi-Tenant Application Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html) recommends deriving and verifying tenant context rather than trusting identifiers supplied by a client.

## How tenant discovery and IdP routing work

In a common enterprise sign-in pattern, the user enters a work email address. The application looks up the domain in its configured SSO connections and sends the browser to the matching IdP. This approach is commonly called *home realm discovery*. Descope also documents optional email-domain and SSO-domain **auto-association** settings that can establish tenant membership; these settings must not be confused with application resource authorization. See [project-level SSO settings](https://docs.descope.com/auth-methods/sso/settings) and [tenant configuration](https://docs.descope.com/management/tenant-management/tenant).

Descope supports tenant-level SSO connections using SAML or OIDC. A tenant can be identified by its configured SSO domain or by an explicit tenant identifier or slug. Descope also documents SSO initiated through [Flows](https://docs.descope.com/auth-methods/sso/with-flows) and through [backend SDKs](https://docs.descope.com/auth-methods/sso). The appropriate integration depends on whether Descope manages the application session or the application maintains its own session after verifying the SSO result.

Email-domain discovery is convenient when one domain reliably points to one connection. It becomes less useful when an organization has several IdPs, a user accesses several customer workspaces, or multiple tenants are configured with the same SSO domain. In those cases, the application needs an intentional routing choice rather than an arbitrary first match.

### Choose a routing method that matches the tenant model

| Configuration | Routing approach | Required follow-up |
| --- | --- | --- |
| One tenant, one unambiguous SSO domain | Match the configured SSO domain | Verify the completed authentication and permitted tenant access |
| Intended workspace known from a branded URL or invitation | Supply a tenant identifier to the authentication flow | Treat that identifier as a selector, not proof of membership |
| One tenant with several SSO providers | Select the applicable connection, including its `ssoId` when needed | Confirm the selected connection's mapping and session behavior |
| Several tenants configured with a shared SSO domain | Resolve the intended tenant explicitly before proceeding | Confirm the actual Flow behavior and post-login membership before granting access |

Descope's [Multiple SSO Providers Per Tenant](https://docs.descope.com/sso/multi-sso) documentation distinguishes **one tenant with several IdPs** from **several tenants each with their own IdP**. Additional SSO profiles have their own `ssoId`, and connection-level SSO domains can help select a provider. If the application already knows which connection is required, it can select that connection explicitly rather than relying on email-domain inference.

The [SSO with Flows](https://docs.descope.com/auth-methods/sso/with-flows) guide describes the project setting **Allow Duplicate SSO Domains Across Tenants** and a Flow condition using `ssoDomainsTenantsCount` to offer tenant selection before SSO. The exact unauthenticated picker contents were not tested in our lab; do not assume the component hides customer names or limits the list to an authenticated user’s memberships. In a production design, limit unnecessary disclosure of customer details during unauthenticated discovery.

There is another distinction worth making: an **SSO Authenticate Only** action validates the federated identity without creating a Descope user or issuing a Descope application session. A full SSO action can establish an application session. When troubleshooting a successful IdP exchange that did not result in login, check which action and connection were used, not just whether the browser returned successfully. Descope describes both actions in its [Flow documentation](https://docs.descope.com/auth-methods/sso/with-flows).

## Authentication succeeds. Is the tenant authorized?

Not necessarily. A correct IdP response proves something about authentication. It does not establish that every resource named in the resulting request belongs to a tenant the caller may use.

Consider a protected API route:

```text
GET /tenants/{tenantId}/invoices/{invoiceId}
```

The browser controls the values submitted for `tenantId` and `invoiceId`. An API that validates a session but then retrieves an invoice solely by `invoiceId` may return another customer's record. A tenant-scoped authorization check must confirm that the caller is permitted to act in the requested tenant and that the invoice belongs to that tenant.

A useful request-handling sequence is:

1. **Validate the session and intended audience.** Use the appropriate backend validator and enforce the expected signature, issuer, lifetime, and authorization policy. **Audience checking is not automatic in Descope’s basic Node.js `validateSession(token)` call**: when supported, supply an explicit `audience` option, such as `validateSession(token, { audience: "__ProjectID__" })` for the documented default Project ID audience, or the intended configured custom audience. Validate other SDKs according to their documented capabilities. [Backend session validation](https://docs.descope.com/sessions/validation/backend), [Node.js quickstart](https://docs.descope.com/getting-started/react/nodejs), and [RFC 8725](https://www.rfc-editor.org/rfc/rfc8725) describe these checks. Offline signature/claim validation alone does not establish whether a previously issued session was subsequently revoked.
2. **Resolve trusted identity and tenant context.** Determine which authenticated principal made the request and which tenant is active or otherwise authorized. A client-supplied `tenantId` is only a requested scope.
3. **Enforce the operation's permissions.** Require the role or permission needed for the action *in that tenant*. A tenant-scoped administrator role must not silently authorize another tenant; any deliberately project-wide role must be evaluated as such.
4. **Constrain the resource lookup.** Fetch or update records under the verified tenant scope; use database-level isolation controls where appropriate as defense in depth.
5. **Deny safely and log the outcome.** Return an appropriate non-revealing response for inaccessible resources and retain sanitized evidence for troubleshooting.

This is a design sequence, not executable Descope SDK code. It applies whether Descope manages the login session or the application creates its own session after federated authentication.

### Active tenant versus available memberships

A user may legitimately belong to several organizations. Descope documents tenant-aware session behavior, including the `dct` (Descope Current Tenant) claim and a `tenants` membership representation in supported user JWT formats. The exact claims available depend on the configured [JWT template](https://docs.descope.com/management/token/jwt-templates); do not assume every token includes the same membership or permission fields.

The application therefore needs an explicit authorization rule: must a protected request match the currently active tenant, or does a particular endpoint permit another membership after an additional server-side check? The secure implementation should make that choice consciously and consistently. Switching workspaces in the user interface must never carry an administrator role from one tenant into another.

OWASP also recommends considering tenant context outside ordinary API queries. Cache entries, background jobs, and service-to-service operations must preserve the correct ownership boundary. A correctly validated JWT cannot compensate for a cache keyed only by a resource ID when that ID is not globally tenant-safe.

## Account linking: why matching email is not enough

A common enterprise migration starts with local sign-in and later adds SSO. A returning user may authenticate through a corporate IdP using an email already associated with a password or one-time-passcode account. Should the two records become one identity?

Only according to a verified account-linking policy. For OIDC, the upstream issuer and subject pair (`iss` and `sub`) is the stable external user identifier defined by [OpenID Connect Core §5.7](https://openid.net/specs/openid-connect-core-1_0.html#ClaimStability). Email is not a substitute: it can change, be reassigned, or be asserted under a different IdP's administrative control. That OIDC principle does not, by itself, define Descope's application-user conversion behavior.

Descope documents controls for [conversion of existing users](https://docs.descope.com/auth-methods/sso/settings) and explains the [risks of merging SSO and non-SSO identities](https://docs.descope.com/sso/merging-sso-identities-risk). Descope’s project-level **Convert Existing Users to SSO-Only** setting and single-tenant conversion safeguard determine when an incoming SSO identity may be merged with an existing user. The [settings guide](https://docs.descope.com/auth-methods/sso/settings) describes conversion as enabled by default, while the [identity-merging guide](https://docs.descope.com/sso/merging-sso-identities-risk) says identities remain separate by default; verify the deployed setting rather than treating either generalization as universal. An ineligible merge can result in a separate user, not necessarily a rejected login. We have not tested the cross-tenant same-email collision or its resulting tenant memberships in our lab.

For an application team, the practical requirements are clear even without predicting every platform edge case:

- Identify which SSO connection is trusted for the tenant.
- Review the deployed conversion and tenant-association settings before migrating existing accounts.
- Confirm which tenant memberships and roles belong to the resulting application identity.
- Ensure a newly created or converted identity cannot inherit another customer's permissions merely because an email matches.

The same distinction applies to SSO enforcement. Descope’s tenant-level **SSO Enabled** setting enforces SSO independently of Flow routing; it differs from the [`ssoEnabled` and `tenant.enforceSSO` conditions](https://docs.descope.com/flows/conditions/sso-enforced) that a Flow uses to choose a path. The [tenant settings guide](https://docs.descope.com/management/tenant-management/tenant) documents an exclusion list and warns that enforcement without a configured SSO domain can prevent users from reaching the IdP. Check both the actual tenant enforcement setting and the Flow behavior; removing enforcement may reopen supported non-SSO sign-in methods. Treat authorization and credential lifecycle separately: forcing SSO at login does not itself prove that previously issued application credentials have been revoked.

## Five security checks before rollout

Rather than attempting to reproduce every possible tenant configuration, validate a small set of failure modes tied directly to the architecture.

| Check | Successful result | Failure that must not occur |
| --- | --- | --- |
| **Routing** | A known tenant or configured domain selects its intended SSO connection | An ambiguous or unmatched domain silently selects the wrong customer |
| **Authentication** | The intended SSO policy and exclusions are enforced; the provider returns the expected identity and session outcome | A required-SSO tenant accepts an unapproved fallback, or an authentication-only result is mistaken for an application session |
| **Tenant authorization** | The API checks the principal, tenant, operation, and resource ownership | Editing tenant or resource IDs exposes another tenant's data |
| **Workspace switching** | A permitted tenant change selects the intended tenant-specific roles (or a separately verified access context) | A role from one tenant carries over to another |
| **Account migration** | Record resulting user ID, tenant memberships and roles after a configured conversion or separate-account outcome | A same-email identity acquires another tenant’s roles or access without authorization |

Run positive tests as well as denials. For example, a user with read-only access to one tenant should successfully read a permitted record but fail an administrative operation; the same account may hold different privileges in another tenant. Validate that the resource layer honors the distinction, not merely that the login screen selected the right IdP.

Use nonproduction identities when testing. Record the selected connection, the authenticated user identity, the tenant context, and the backend authorization result. Do not save real tokens, secrets, or unnecessary personally identifying data in screenshots or published logs.

## Where Descope fits into the design

Descope provides tenant-based SSO configuration, SAML and OIDC connections, routing through Flows or SDKs, tenant-aware session capabilities, and settings for SSO enforcement and identity handling. Those features reduce how much federation and account-management infrastructure a B2B application must implement directly. They do not eliminate the application's responsibility to protect its own tenant-owned data.

For a team adding enterprise SSO, the most useful starting points are the [SSO overview](https://docs.descope.com/auth-methods/sso), [SSO with Flows](https://docs.descope.com/auth-methods/sso/with-flows), [multiple providers per tenant](https://docs.descope.com/sso/multi-sso), and [backend session validation](https://docs.descope.com/sessions/validation/backend).

A limited integration can validate part of this architecture without establishing all of it. In an independent Keycloak–Descope lab, we completed one OIDC sign-in through a tenant-specific Flow, observed a Descope session response, and confirmed Just-in-Time creation of an active tenant-associated user. The [public case study](https://github.com/michael-k-levesque/keycloak-descope-oidc-documentation) documents that observation. **All additional multi-tenant routing, multiple-provider, authentication-only, SSO enforcement, identity-conversion, switching, and resource-authorization behavior described here is sourced from documentation or presented as engineering guidance, not as tested outcomes from our lab.**

## Frequently asked questions

### Does an email domain prove which tenant a user can access?

No. A configured domain can help select an SSO route and may automatically associate a user with a tenant, but it does not independently establish the permissions required to access a particular application resource.

### Can one tenant use more than one identity provider?

Yes. Descope documents several SSO configurations under one tenant, with separate connection identifiers and routing considerations. This differs from creating a separate tenant for each customer.

### Does a valid Descope session authorize every resource under the current tenant?

No. A session establishes authenticated context. The API must still enforce permissions for the requested operation and verify ownership of the target resource.

### Should two accounts with the same email automatically be linked?

Not as a general application rule. Review the configured conversion and association settings, external identity trust, and resulting tenant permissions. Email similarity alone is not a secure cross-tenant authorization decision.

## Final takeaway

Tenant discovery determines **where authentication should begin**. Federation establishes **which identity authenticated**. Tenant authorization determines **what that identity may do** within the application.

A secure multi-tenant SSO design keeps those responsibilities separate. Select the right connection, verify the authentication result, and enforce tenant ownership on every protected operation. That is the difference between a login flow that works and a B2B application that keeps its customers' data isolated.

## Technical references

- [Descope — SSO authentication overview](https://docs.descope.com/auth-methods/sso)
- [Descope — SSO with Flows](https://docs.descope.com/auth-methods/sso/with-flows)
- [Descope — Multiple SSO Providers Per Tenant](https://docs.descope.com/sso/multi-sso)
- [Descope — Project-Level SSO Settings](https://docs.descope.com/auth-methods/sso/settings)
- [Descope — Tenant Configuration and SSO Enforcement](https://docs.descope.com/management/tenant-management/tenant)
- [Descope — Identity Merging Risks](https://docs.descope.com/sso/merging-sso-identities-risk)
- [Descope — User JWT Templates](https://docs.descope.com/management/token/jwt-templates)
- [Descope — Backend Session Validation](https://docs.descope.com/sessions/validation/backend)
- [OWASP — Multi-Tenant Application Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html)
- [OpenID Connect Core 1.0, Claim Stability (§5.7)](https://openid.net/specs/openid-connect-core-1_0.html#ClaimStability)
- [RFC 8725 — JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725)