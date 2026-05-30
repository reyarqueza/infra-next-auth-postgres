---
name: vercel-custom-domain
description: >-
  Add a predictable custom domain to the Vercel project and create the Cloudflare
  DNS record. Invoked by bootstrap orchestrator only.
disable-model-invocation: true
---

# Vercel custom domain

Wire `{APP_NAME}.{DOMAIN_NAME}` to the Vercel project so `PRODUCTION_URL` is configured before auth env setup. HTTP verification is deferred to [verify-deploy](../verify-deploy/SKILL.md).

Parameters: `APP_NAME`, `DOMAIN_NAME`, `VERCEL_PROJECT_NAME`, `VERCEL_TEAM`, `LOCAL_PATH`, `PRODUCTION_URL`, `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ZONE_ID`.

Derived:

```bash
CUSTOM_HOSTNAME="${APP_NAME}.${DOMAIN_NAME}"
# PRODUCTION_URL must equal https://${CUSTOM_HOSTNAME}
```

Requires `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ZONE_ID` in the environment (see [preflight-auth](../preflight-auth/SKILL.md)).

## Resolve zone ID (if missing)

If `CLOUDFLARE_ZONE_ID` is unset, look it up once:

```bash
CLOUDFLARE_ZONE_ID="$(curl -sf -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  "https://api.cloudflare.com/client/v4/zones?name=${DOMAIN_NAME}" \
  | jq -r '.result[0].id // empty')"
test -n "${CLOUDFLARE_ZONE_ID}"
```

## Add domain to Vercel project

From the linked app directory:

```bash
cd "${LOCAL_PATH}"
vercel domains add "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}" </dev/null
```

If the domain is already on the account, link it to this project:

```bash
vercel project domain add "${CUSTOM_HOSTNAME}" "${VERCEL_PROJECT_NAME}" --scope "${VERCEL_TEAM}" 2>/dev/null \
  || vercel domains add "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}" </dev/null
```

## Cloudflare DNS record

Use DNS-only (grey cloud) so Vercel can serve the domain ([Vercel + Cloudflare](https://vercel.com/docs/integrations/external-platforms/cloudflare#using-cloudflare-as-your-dns-provider)).

### Prefer A record for subdomains (Cloudflare + Vercel)

When Vercel recommends an A record for `{APP_NAME}.{DOMAIN_NAME}` (common for subdomains on Cloudflare-managed zones), use:

```bash
VERCEL_A_TARGET="76.76.21.21"
```

Check for an existing A record:

```bash
EXISTING_ID="$(curl -sf -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records?type=A&name=${CUSTOM_HOSTNAME}" \
  | jq -r '.result[0].id // empty')"
```

Create if missing:

```bash
curl -sf -X POST "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data "{\"type\":\"A\",\"name\":\"${APP_NAME}\",\"content\":\"${VERCEL_A_TARGET}\",\"proxied\":false,\"ttl\":1}"
```

Update if present and content differs:

```bash
curl -sf -X PUT "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records/${EXISTING_ID}" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data "{\"type\":\"A\",\"name\":\"${APP_NAME}\",\"content\":\"${VERCEL_A_TARGET}\",\"proxied\":false,\"ttl\":1}"
```

### CNAME fallback

If `vercel domains inspect` shows a project-specific CNAME target instead of an A record recommendation, use CNAME:

```bash
VERCEL_CNAME_TARGET="$(vercel domains inspect "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}" 2>/dev/null \
  | grep -i 'cname' | head -1 | awk '{print $NF}')"
```

Create or update the CNAME for `{APP_NAME}` pointing at `VERCEL_CNAME_TARGET` (same create/update pattern as A record, with `"type":"CNAME"`).

## Optional quick check (do not block)

Optionally confirm DNS is propagating — **cap at 2 attempts, 30 seconds apart**. Do not poll for 10 minutes.

```bash
for i in 1 2; do
  curl -sf -o /dev/null -w "%{http_code}" --max-time 10 "https://${CUSTOM_HOSTNAME}/" && break
  sleep 30
done
```

Continue bootstrap regardless of HTTP result. Full smoke checks run in [verify-deploy](../verify-deploy/SKILL.md).

## Gate

- Domain added to the Vercel project (`vercel domains add` succeeded)
- Cloudflare DNS record exists for `{APP_NAME}` (A → `76.76.21.21` or CNAME → Vercel target)
- `PRODUCTION_URL` is `https://${CUSTOM_HOSTNAME}`

**Do not block** on Vercel domain verification or HTTP 200 from the custom hostname. Proceed to step 6 (or parallel step 6) once the DNS record exists.
