---
name: vercel-custom-domain
description: >-
  Add a predictable custom domain to the Vercel project and create the Cloudflare
  DNS CNAME record. Invoked by bootstrap orchestrator only.
disable-model-invocation: true
---

# Vercel custom domain

Wire `{APP_NAME}.{DOMAIN_NAME}` to the Vercel project so `PRODUCTION_URL` is reachable before auth env setup and deploy verification.

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
vercel domains add "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}"
```

If the domain is already on the account, link it to this project:

```bash
vercel project domain add "${CUSTOM_HOSTNAME}" "${VERCEL_PROJECT_NAME}" --scope "${VERCEL_TEAM}" 2>/dev/null \
  || vercel domains add "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}"
```

## Read Vercel CNAME target

Vercel assigns a project-specific CNAME for subdomains. Inspect the domain:

```bash
VERCEL_CNAME_TARGET="$(vercel domains inspect "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}" 2>/dev/null \
  | grep -i 'cname' | head -1 | awk '{print $NF}')"
```

If empty, parse JSON:

```bash
VERCEL_CNAME_TARGET="$(vercel domains inspect "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}" --json 2>/dev/null \
  | jq -r '.. | strings | select(test("\\.vercel-dns"))' | head -1)"
```

Stop if `VERCEL_CNAME_TARGET` is empty — re-run inspect and show output to the user.

## Create or update Cloudflare CNAME

Use DNS-only (grey cloud) so Vercel can verify the domain ([Vercel + Cloudflare](https://vercel.com/docs/integrations/external-platforms/cloudflare#using-cloudflare-as-your-dns-provider)).

Check for an existing record:

```bash
EXISTING_ID="$(curl -sf -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records?type=CNAME&name=${CUSTOM_HOSTNAME}" \
  | jq -r '.result[0].id // empty')"
```

Create if missing:

```bash
curl -sf -X POST "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data "{\"type\":\"CNAME\",\"name\":\"${APP_NAME}\",\"content\":\"${VERCEL_CNAME_TARGET}\",\"proxied\":false,\"ttl\":1}"
```

Update if present and content differs:

```bash
curl -sf -X PUT "https://api.cloudflare.com/client/v4/zones/${CLOUDFLARE_ZONE_ID}/dns_records/${EXISTING_ID}" \
  -H "Authorization: Bearer ${CLOUDFLARE_API_TOKEN}" \
  -H "Content-Type: application/json" \
  --data "{\"type\":\"CNAME\",\"name\":\"${APP_NAME}\",\"content\":\"${VERCEL_CNAME_TARGET}\",\"proxied\":false,\"ttl\":1}"
```

## Wait for Vercel verification

Poll until the domain is configured (up to 10 minutes, every 30 seconds):

```bash
for i in $(seq 1 20); do
  STATUS="$(vercel domains inspect "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}" 2>/dev/null | grep -i 'verified\|configured\|valid' || true)"
  vercel domains inspect "${CUSTOM_HOSTNAME}" --scope "${VERCEL_TEAM}" 2>/dev/null | grep -qi 'invalid\|pending' || { echo "Domain ready"; break; }
  sleep 30
done
```

Alternatively confirm the hostname resolves and Vercel serves the project:

```bash
curl -sf -o /dev/null -w "%{http_code}" "https://${CUSTOM_HOSTNAME}/" || true
```

DNS propagation may take a few minutes; continue bootstrap once Vercel shows the domain as verified or `curl` returns a non-connection error (3xx/4xx/5xx).

## Gate

- Cloudflare CNAME exists for `{APP_NAME}` pointing at the Vercel-assigned target
- `PRODUCTION_URL` is `https://${CUSTOM_HOSTNAME}`
- Vercel domain inspect no longer reports invalid/misconfigured DNS (or HTTP check to `PRODUCTION_URL` succeeds)

Do not proceed to marketplace Neon until the gate passes — auth env and verify-deploy depend on the custom hostname.
