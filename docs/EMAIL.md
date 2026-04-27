# Email runbook

> See also: [INFRASTRUCTURE.md](./INFRASTRUCTURE.md).

Email for `@berensofware.com` runs on **Cloudflare Email Routing** (free, no third-party). It is **inbound-only**: messages addressed to `info@` or `support@` are forwarded to a Gmail mailbox. Sending mail _as_ `info@berensofware.com` requires an outbound SMTP relay (not yet wired — see end of doc).

## Active rules

| Match | Forward to | Status |
|---|---|---|
| `info@berensofware.com` | `berensoftware@gmail.com` | Enabled |
| `support@berensofware.com` | `berensoftware@gmail.com` | Enabled |
| `*` (catch-all anything else) | Drop | Enabled |

## Resource IDs (Cloudflare)

| Resource | ID |
|---|---|
| Account | `03eb1f70d324e573bde2ca54eefe0fb9` |
| Zone (`berensofware.com`) | `ace7f2d09ad3421ab99de669618e7394` |
| Destination address `berensoftware@gmail.com` | tag `8e28962f39f04b109c8ed41e05b539a0` |
| Rule `info` | tag `22a59d77349341c69554fb262fd6398b` |
| Rule `support` | tag `3914af2833fb49db846d18f5656f7338` |

The tag is the URL-safe ID Cloudflare assigns. Use it in API URLs like `/zones/:zone_id/email/routing/rules/:tag`.

## DNS records that make it work

All on Cloudflare DNS for `berensofware.com`:

| Type | Name | Content | Priority |
|---|---|---|---|
| MX | `@` | `route1.mx.cloudflare.net` | 49 |
| MX | `@` | `route2.mx.cloudflare.net` | 85 |
| MX | `@` | `route3.mx.cloudflare.net` | 71 |
| TXT | `@` | `v=spf1 include:_spf.mx.cloudflare.net ~all` | — |
| TXT | `cf2024-1._domainkey` | `v=DKIM1; h=sha256; k=rsa; p=…` | — |

The DKIM record is auto-managed by Cloudflare; do not edit by hand. Cloudflare rotates the key periodically.

## Common operations

### Add a new alias (e.g. `hr@`)

Via the Cloudflare API with the project-scoped token:

```bash
TOKEN="<beren-software-deploy token>"
ZONE_ID="ace7f2d09ad3421ab99de669618e7394"

curl -X POST \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name":"hr",
    "enabled":true,
    "matchers":[{"type":"literal","field":"to","value":"hr@berensofware.com"}],
    "actions":[{"type":"forward","value":["berensoftware@gmail.com"]}]
  }'
```

Or via dashboard: Cloudflare → `berensofware.com` → Email → Email Routing → Routing rules → Create address.

### Add a different forward target

Cloudflare requires every destination address to be _verified_ before it can be used in a rule. Two-step:

1. Add destination (triggers a verification email):

```bash
ACCOUNT_ID="03eb1f70d324e573bde2ca54eefe0fb9"
curl -X POST \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/email/routing/addresses" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"email":"new-target@example.com"}'
```

2. Open the inbox of `new-target@example.com`, click the verification link Cloudflare sent.

3. Confirm verification status:

```bash
curl -s "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/email/routing/addresses" \
  -H "Authorization: Bearer $TOKEN" | python -m json.tool
```

`verified` field becomes a timestamp once done. Now the address can be used as `actions[].value` in any rule.

### Update an existing rule (change forward target, rename, etc.)

```bash
ZONE_ID="ace7f2d09ad3421ab99de669618e7394"
RULE_TAG="22a59d77349341c69554fb262fd6398b"   # info

curl -X PUT \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/email/routing/rules/$RULE_TAG" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name":"info",
    "enabled":true,
    "matchers":[{"type":"literal","field":"to","value":"info@berensofware.com"}],
    "actions":[{"type":"forward","value":["new-target@example.com"]}]
  }'
```

PUT replaces the whole rule, so include all fields you want to keep.

### Delete a destination

```bash
ACCOUNT_ID="03eb1f70d324e573bde2ca54eefe0fb9"
TAG="<destination tag>"
curl -X DELETE \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/email/routing/addresses/$TAG" \
  -H "Authorization: Bearer $TOKEN"
```

You cannot delete a destination that is still referenced by an active rule. Update the rule first.

### Pause routing

Disable individual rules with `"enabled": false` in the PUT body. To pause routing entirely, disable Email Routing from the dashboard (this requires `Email Routing Settings:Edit` which the current API token does not have — recreate token if needed).

## Verification & troubleshooting

### Look at the Activity Log

`https://dash.cloudflare.com/<account>/berensofware.com/email/routing/overview` → Activity Log shows every inbound message with status (Forwarded / Rejected / Failed).

### A test message disappears

Most likely causes:

1. **DNS cache**: public resolvers (1.1.1.1, 8.8.8.8) sometimes hold stale MX records for ~10 minutes after a switchover. New senders may resolve to the old MX and bounce. Wait, retry.
2. **Gmail spam classification**: Cloudflare-routed mail can land in **Promozioni / Aggiornamenti** in Gmail when the sender isn't a known contact. Check those tabs first.
3. **DKIM/SPF mismatch on the sender side**: Cloudflare relays the message _as-is_; if the original sender's domain has misconfigured DKIM, Gmail may quarantine. Cloudflare's own DKIM (`cf2024-1._domainkey`) is for messages _originating_ from forwarding (rare) and is auto-managed.
4. **Gmail filter rules**: check `https://mail.google.com/mail/u/0/#settings/filters`.

### Nuclear option: re-test

```bash
# Send a test from any account (e.g. via Gmail compose) to:
#   info@berensofware.com
#   support@berensofware.com
# Then refresh the Cloudflare Activity Log; messages should appear within seconds.
```

## Outbound (Send-as) — not yet configured

To reply from Gmail showing `info@berensofware.com` as sender, configure Gmail's "Send mail as" feature with an outbound SMTP relay. Free options:

| Service | Free tier | Notes |
|---|---|---|
| Brevo (formerly Sendinblue) | 300 mail/day | Most generous free tier; gives SMTP creds + dashboard |
| SendGrid | 100 mail/day | Mature, easy Gmail integration |
| Resend | 3000 mail/month | Developer-friendly; SMTP available |
| MailerSend | 3000 mail/month | Includes domain verification flow |

**Generic flow** for any of them:
1. Sign up, verify domain by adding a TXT record to Cloudflare DNS (5 min)
2. Generate SMTP credentials (host, port 587, user, pass) and a sender like `info@berensofware.com`
3. Gmail → Settings → Accounts → "Send mail as" → Add another email address → tick "Treat as alias" → enter SMTP creds
4. Gmail sends a confirmation code to `info@berensofware.com`; it arrives via Cloudflare Email Routing in `berensoftware@gmail.com` inbox; paste back
5. Done. Compose in Gmail, switch From → `info@berensofware.com`, send

This step is deferred — say the word and it can be wired up.

## DMARC (also deferred)

Recommended record once email volume grows:

```
TXT  _dmarc.berensofware.com  "v=DMARC1; p=none; rua=mailto:berensoftware@gmail.com; pct=100;"
```

Add via:

```bash
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"type":"TXT","name":"_dmarc","content":"v=DMARC1; p=none; rua=mailto:berensoftware@gmail.com; pct=100;","ttl":1}'
```

`p=none` collects aggregate reports without rejecting any mail — safe to start.
