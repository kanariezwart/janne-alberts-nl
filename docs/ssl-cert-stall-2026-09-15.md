# SSL certificate stall after NS cutover (2026-09-15)

## Symptom

`janne-alberts.nl` and `www.janne-alberts.nl` returned
`ERR_SSL_VERSION_OR_CIPHER_MISMATCH` in the browser after nameservers were
switched to Cloudflare. `curl`/`openssl s_client` showed a TLS
`handshake_failure` (alert 40) — Cloudflare's edge had no certificate to
present for the hostname's SNI.

## Cause

Attaching the two Workers Custom Domains (`janne-alberts.nl` and
`www.janne-alberts.nl`) had each auto-created their own dedicated Advanced
Certificate Manager cert pack (Google CA), on top of the zone's own
Universal SSL cert (Let's Encrypt). All three overlapping cert orders sat
in `pending_validation` for over an hour, even though the `_acme-challenge`
TXT records were already publicly correct (verified via 8.8.8.8 and
Cloudflare's own nameservers directly) — so this wasn't a DNS problem, just
a stuck/queued CA order.

## Fix

1. Deleted the two redundant Advanced cert packs
   (`DELETE /zones/{zone_id}/ssl/certificate_packs/{id}`).
2. A `PUT` to `/accounts/{account_id}/workers/domains` with the same
   hostname was **not** enough to force reissue — it just returned the
   same stale `cert_id`.
3. Deleted both Workers Custom Domain bindings entirely
   (`DELETE /accounts/{account_id}/workers/domains/{id}`), then recreated
   them with the same `PUT`. This produced new `cert_id`s and triggered
   fresh cert orders.
4. The zone's Universal SSL cert (`*.janne-alberts.nl`) went `active`
   shortly after step 1 — its wildcard coverage already serves both the
   apex and `www` correctly, independent of the two Advanced certs still
   finishing in the background.

## Takeaway for next time

If certificate issuance stalls on a Workers Custom Domain after an NS
cutover, don't just wait it out — a full delete + recreate of the Custom
Domain binding forces a genuinely new cert order, whereas re-`PUT`-ing the
same binding is a no-op.
