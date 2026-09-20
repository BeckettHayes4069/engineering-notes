# DNS or Service Registry for Internal Endpoints — Deploy Drift in Gaming Onboarding

Short answer: use public DNS to prove control of a gaming partner's domain, and use an internal registry only for locating rapidly replaced application instances. A registry entry can describe where a service runs; it cannot prove that a partner controls the public domain named in an onboarding request. The deciding constraint is drift: the TXT value the partner intended to publish may differ from what a resolver can actually retrieve, while an internal endpoint can become stale after a deployment even when the domain proof remains valid.

For a one-person SaaS shipping weekly, those are two different clocks. Trying to make one naming system solve both spends engineering hours on the wrong failure mode.

## What does a domain challenge actually prove?

An onboarding request might claim `arcade.example` and ask the operator of that domain to publish a unique TXT value at `_onboard.arcade.example`. The service should issue a fresh, unpredictable token, store the expected owner and exact record name, and query DNS for the published value before marking that request verified. A DNS response establishes that the value is visible at the queried name; it does not identify the human who changed the zone or grant permanent authority over future changes. Set an expiry on the challenge and decide separately when ownership needs rechecking.

Keep the claimed domain in the request distinct from any hostname the gaming application uses to reach a match coordinator. A successful TXT check should never overwrite an internal endpoint. A failed check should leave onboarding incomplete, with the expected name and a safe-to-display status available to the operator.

No token matches? No verification.

DNS caching matters here. RFC 1035 defines TTLs for resource records, and RFC 2308 covers negative caching for answers such as a missing record. A partner can add a TXT record and still encounter a cached earlier absence. Consider the sequence: the verifier looks before the partner publishes, receives a negative answer, and then looks again after the zone has changed. The second lookup can still report absence if its resolver retained the negative response. The application must distinguish that pending state from an expired challenge, even though both leave onboarding unfinished. Likewise, removing a value from an authoritative zone does not imply every recursive resolver immediately stops returning the earlier answer. Treat a pending result as pending, retry with a bounded deadline, and do not claim that shortening the new record's TTL flushes a previously cached negative answer.

## The smallest useful verification loop

The example below performs one TXT lookup and checks for the entire expected value. It deliberately makes no claim that a single lookup is enough to establish global propagation. In production, issue the token with a cryptographic random generator, associate it with one pending onboarding request, expire it, rate-limit checks, and record the queried name and check time.

```ts
import { resolveTxt } from 'node:dns/promises';

type Check = 'verified' | 'pending' | 'retry';

export async function checkOwnership(
  recordName: string,
  expectedValue: string,
): Promise<Check> {
  try {
    const records = await resolveTxt(recordName);
    return records.some((parts) => parts.join('') === expectedValue)
      ? 'verified'
      : 'pending';
  } catch (error) {
    if (error instanceof Error && 'code' in error) {
      if (error.code === 'ENOTFOUND' || error.code === 'ENODATA') {
        return 'pending';
      }
    }
    return 'retry';
  }
}
```

TXT data can be split into multiple character strings within one record; joining the parts of each returned record preserves the value before comparison. Do not concatenate separate records or accept a substring. The DNS lookup API reports what its configured resolver sees, so a pending result is evidence about that view, not a verdict on the partner's intent. A timeout or resolver failure takes the `retry` path rather than becoming a false ownership denial.

There is a practical security boundary: never build the queried name by accepting an arbitrary caller-supplied hostname and blindly querying it. Normalize and validate the claimed domain, bind the challenge to an authenticated onboarding request, and constrain the record name to that domain. The verification service should also use a bounded retry policy. Otherwise a stalled DNS check can become a stalled onboarding worker.

## Should DNS or a service registry resolve internal endpoints after a deploy?

Once the domain is verified, the game still needs to find an internal coordinator. If instances rotate during frequent deployments, discovery must reflect the set of healthy, eligible instances rather than the partner's domain ownership. A registry can track membership and health; DNS can also serve internal discovery if its publication and caching behavior meet the deployment's freshness requirement. Neither choice eliminates the need for client-side timeouts, connection draining, and retry rules that avoid duplicating non-idempotent operations.

The useful measurement is not deploys per day by itself. Compare the interval between withdrawing an instance and clients ceasing to send it traffic with the time that instance stays available for draining. Instrument stale-target connection failures and lookup errors separately from TXT challenge outcomes. This makes intent-versus-publication drift visible: an intended registry update is not proof that clients have adopted it, just as a saved TXT value is not proof that resolvers see it.

A weekly release rhythm favors outsourcing undifferentiated mechanics where possible, but that is a time-allocation rule, not a product endorsement.

Budget engineering hours for the verification state machine and failure signals before tuning discovery infrastructure.

## What changes when traffic grows?

At scale, add independent checks from the resolver paths your users actually depend on, and document how disagreement affects pending requests. Keep challenge issuance, expiry, and successful verification auditable. For discovery, test a rolling replacement with an old instance still serving, then repeat with it unavailable; observe how long clients keep targeting the old address. These tests expose different faults and should not share a single green dashboard light.

The trade-off stays plain. DNS-based discovery may be sufficient when endpoint lifetimes and cache behavior fit the drain window. A registry becomes useful when instance membership and health must change faster than that contract permits, but it adds a control plane and client integration to operate. Keep public domain proof separate in either design. That boundary lets onboarding proceed on evidence from the published record while deployment frequency dictates only the internal routing mechanism.

## References

- https://www.rfc-editor.org/rfc/rfc1035
- https://www.rfc-editor.org/rfc/rfc2308
- https://nodejs.org/api/dns.html#dnspromisesresolvetxthostname
