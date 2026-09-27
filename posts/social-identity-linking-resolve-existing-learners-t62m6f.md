# Social Identity Linking: Resolve Existing Learners Before Creating Duplicate Accounts

TL;DR: Resolve the social identity and any verified local identifier before creating a learner. Put CAPTCHA in front of the new-account branch only. If a provider identity already belongs to a user, sign in that user; if a verified email points to an existing user, require that user's normal reauthentication or recovery path before linking; otherwise create the account and identity link in one transaction. Never merge accounts merely because two profile payloads contain the same email.

| Incoming state | Decision | CAPTCHA | Recovery consequence |
|---|---|---|---|
| Provider identity already linked | Sign in the linked user | No new-signup challenge | Existing recovery methods remain authoritative |
| No link, verified identifier matches one user | Pause and prove control of that user | Do not create yet | Recovery or reauthentication authorizes the link |
| No link, no verified match | Verify CAPTCHA, then create user and link | Required | Establish a usable recovery path |
| Match is missing, ambiguous, or unverified | Stop for explicit resolution | Never auto-merge | Preserve both accounts until ownership is proved |

**Recommendation:** make identity resolution a deterministic step before account creation, backed by unique database constraints and a transaction. This costs more design time than a find-or-create helper, but it prevents the expensive case: a learner gets course progress, classroom membership, or guardian approval split across two accounts. For a small edtech service shipping weekly, that support burden steals revenue-producing hours.

## How should Node.js link a social identity to an existing user?

An email-shaped string is a hint, not proof that the current person controls an existing account. Treat the social provider's stable subject identifier and issuer as the external identity key. Keep that pair separate from the local user record: one user may have several login methods, while each external identity may belong to only one local user.

The dangerous shortcut looks harmless. Receive a social callback, search by email, and attach the identity to the first row. It collapses identity discovery and authorization into one operation. It also makes recovery unpredictable because a later login can silently change which credential reaches old coursework.

Email is not authorization.

OWASP recommends reauthentication for sensitive account changes and calls for secure account recovery. Linking a new sign-in method changes who can enter the account, so it belongs in that category. A currently authenticated session should reauthenticate before adding a social identity. A signed-out person who collides with an existing verified identifier should complete the established recovery flow, then confirm the link from that recovered account.

Keep failure responses dull. A public callback should not reveal whether a learner email exists. Store the pending link server-side under a short-lived, single-use nonce, then send the person through the same generic recovery entry point. This avoids turning identity linking into an account-discovery tool.

## The two decisions that protect recovery

First ask whether the incoming identity is already known. Query by `(issuer, subject)`, not display name or mutable profile fields. If it exists, perform an ordinary sign-in. Do not rerun CAPTCHA and do not create a learner. A CAPTCHA can challenge automated traffic; it cannot prove ownership of a local account.

Then ask whether a new identity may attach to an existing user. There are two defensible proofs: reauthentication inside that user's authenticated session, or successful completion of that user's recovery path. Require the same assurance used before changing a password or primary email.

Recovery has to work when the learner loses access to the social identity. Before making social login the only credential, decide what the service can legitimately recover: a verified email, a guardian-controlled flow, or an institution-managed route. Do not invent a support bypass. Manual recovery needs an auditable policy and evidence appropriate to the account's impact.

For minors and classrooms, identifiers can be shared, recycled, or managed by another person. The application therefore needs an explicit `needs_resolution` state instead of guessing.

Stop.

Preserving two records temporarily is cheaper than attaching one learner's login to another learner's history.

## A transaction, not a find-or-create helper

The implementation below keeps vendor details behind interfaces. It assumes the callback token has already been validated according to the identity protocol in use. The resolver consumes only a validated principal. CAPTCHA verification occurs on the server, and only the create branch can require it.

```ts
interface SocialPrincipal {
  issuer: string;
  subject: string;
  email?: string;
  emailVerified: boolean;
}

type Resolution =
  | { kind: 'signed_in'; userId: string }
  | { kind: 'linked'; userId: string }
  | { kind: 'created'; userId: string }
  | { kind: 'recovery_required'; pendingLinkId: string }
  | { kind: 'needs_resolution' };

async function resolveSocialIdentity(input: {
  principal: SocialPrincipal;
  captchaToken?: string;
  authenticatedUserId?: string;
  reauthenticated: boolean;
}): Promise<Resolution> {
  const linked = await identities.find(input.principal.issuer, input.principal.subject);
  if (linked) return { kind: 'signed_in', userId: linked.userId };

  if (input.authenticatedUserId) {
    if (!input.reauthenticated) throw new Error('REAUTHENTICATION_REQUIRED');
    await identities.insertUnique({
      userId: input.authenticatedUserId,
      issuer: input.principal.issuer,
      subject: input.principal.subject,
    });
    return { kind: 'linked', userId: input.authenticatedUserId };
  }

  const candidate = input.principal.emailVerified && input.principal.email
    ? await users.findUniqueByNormalizedEmail(input.principal.email)
    : undefined;

  if (candidate) {
    const pendingLinkId = await pendingLinks.createSingleUse({
      userId: candidate.id,
      issuer: input.principal.issuer,
      subject: input.principal.subject,
    });
    return { kind: 'recovery_required', pendingLinkId };
  }

  if (!input.captchaToken) throw new Error('CAPTCHA_REQUIRED');
  if (!await captcha.verify(input.captchaToken)) throw new Error('CAPTCHA_REJECTED');

  return db.transaction(async (tx) => {
    const user = await tx.users.insert({
      verifiedEmail: input.principal.emailVerified ? input.principal.email : undefined,
    });
    await tx.identities.insertUnique({
      userId: user.id,
      issuer: input.principal.issuer,
      subject: input.principal.subject,
    });
    return { kind: 'created', userId: user.id };
  });
}
```

The database must enforce uniqueness for the external key. Application checks alone race: two callbacks can both observe no link, then create records. Use a unique constraint on `(issuer, subject)` and translate its conflict into a fresh lookup. If verified local email is unique in the product model, enforce that too. If it is not, an email search can return several candidates and the resolver must choose `needs_resolution`.

Verify CAPTCHA immediately before entering the transaction, bind success to the signup attempt, and consume it once. Do not hold a database transaction open during a network call. This keeps locks short without allowing a solved challenge to become a reusable creation ticket.

A second concurrency edge appears after recovery. The pending link may have been consumed, expired, or attached elsewhere while the learner was proving control. Complete it with a conditional write that checks single-use status and the identity uniqueness constraint together. On conflict, restart identity resolution. Do not retry account creation blindly.

## When is the cautious branch worth it?

Always use explicit recovery when a verified identifier matches an existing account and there is no authenticated, recently reauthenticated session. The extra screen is friction, but its purpose is visible. A silent merge creates invisible damage.

The runner-up approach is to create a separate account and offer a later, user-initiated merge. It is better when identifiers are commonly shared, the account model permits multiple people behind one address, or the service cannot establish adequate proof during signup. It also fits cases where merging domain data requires review: course enrollments, assessment attempts, certificates, and guardian relationships may have conflicts that a generic authentication transaction cannot settle.

This approach has limits. It is unsuitable when the service has no recovery method independent of the incoming social identity, because the collision branch cannot prove control and will strand the learner. It is also a poor fit when one email intentionally represents several learners; in that model, email cannot be a unique lookup key at all. Choose separate-account creation with a later reviewed merge in those cases. The downside is temporary duplication and more domain reconciliation work, but that trade-off is honest: the system lacks enough evidence to attach the login during signup.

Later merging needs its own rules. Prove control of both accounts, define which record wins for every domain object, keep an audit event, and make the operation idempotent. Authentication rows are the easy part. Data ownership is where irreversible mistakes happen.

For a one-person service, I would outsource CAPTCHA processing and protocol token validation behind narrow adapters, but keep identity resolution and recovery policy in application code. Those rules encode the product's data model. They are differentiated enough to own.

Test the state machine, not just the happy callback. The smallest useful suite covers an existing external link, a verified email collision, an unverified email, an authenticated link without reauthentication, a failed CAPTCHA, concurrent first logins, and reuse of a consumed pending link. Add one domain test showing that a resolution failure cannot move enrollments or progress.

Log decision outcomes without storing raw tokens. Useful fields include a request correlation ID, a pseudonymous user or pending-link identifier, the branch selected, challenge outcome, and unique-constraint conflicts. A rising CAPTCHA pass rate says nothing about whether an account link is authorized; the controls answer different questions.

Keep those signals separate.

**The durable rule is simple:** resolve identity first, authorize linking through reauthentication or recovery second, and create only when neither step finds an account. CAPTCHA protects that final creation branch from automation. It never settles ownership.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
