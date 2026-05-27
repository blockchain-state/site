# ops

Operational artifacts for Math.R infrastructure. Not deployed with the site.

---

## n8n-github-to-kchat.json — GitHub → kChat #mathr-dev

Status: **deployed** in n8n (workflow `GitHub → kChat #mathr-dev`). Currently wired to `blockchain-state/site` repo webhook.

### Planned upgrade (org-level + per-channel routing)

Replace the existing per-repo webhook on `site` with an **org-level webhook** in `blockchain-state` that fires for all repos. Then extend the `Send to kChat` URL to route by `repository.name`:

```
site            → #mathr-dev   (current kChat hook)
math-r-bs       → #bs-protocol  (NEW kChat hook needed)
arq             → #bs-protocol  (NEW kChat hook needed)
mars-protocol   → #mars-protocol (NEW kChat hook needed)
aether-vortex   → #vayu-engineering (NEW kChat hook needed)
```

In `Send to kChat` node, replace static URL with expression:

```javascript
={{ ({
  'site': 'https://.../019e5f9b-c0a0-70fb-bfb0-4fb38c4cb26e',
  'math-r-bs': '<NEW_HOOK_BS_PROTOCOL>',
  'arq': '<NEW_HOOK_BS_PROTOCOL>',
  'mars-protocol': '<NEW_HOOK_MARS>',
  'aether-vortex': '<NEW_HOOK_VAYU>'
})[$json.body.repository.name] || 'https://.../019e5f9b-c0a0-70fb-bfb0-4fb38c4cb26e' }}
```

To complete:
1. Create kChat incoming webhooks for `#bs-protocol`, `#mars-protocol`, `#vayu-engineering`.
2. Update `Send to kChat` URL expression with real values.
3. Add org-level webhook in `blockchain-state` Organization Settings → Webhooks → `https://n8n.mathr.ch/webhook/github-mathr` (events: push, pull_request).
4. Delete the per-repo webhook on `blockchain-state/site` (avoids duplicate messages).
5. Push to any repo → message lands in correct channel.

---

## n8n-intake-to-kchat.json — /entry form → kChat #mathr-dev

Status: **needs import**. New workflow for receiving submissions from `mathr.ch/entry` (EN/RU/FR).

### What it does

1. Receives POST at `/webhook/intake` with form fields: `tension`, `context`, `contact`, `name`, `lang`, `source`, `website` (honeypot).
2. Switch checks honeypot (`website` not empty → drop as spam) and required field (`tension` not empty → format).
3. Formats a markdown message and posts to kChat `#mathr-dev` (same kChat hook as GitHub workflow).
4. Responds with `200 OK` plus CORS headers (`Access-Control-Allow-Origin: https://mathr.ch`).

### Setup

1. **Import**: in n8n, Workflows → ⋮ → Import from File → pick `n8n-intake-to-kchat.json`.
2. **Verify kChat URL** in `Send to kChat #mathr-dev` node matches the production hook (currently hard-coded to the existing `frolenko-oleg.kchat.infomaniak.com/hooks/019e5f9b-...` URL, same as the GitHub workflow).
3. **Activate** the workflow (toggle top-right). Production URL becomes:
   ```
   https://n8n.mathr.ch/webhook/intake
   ```
4. **Test from the browser**: navigate to `https://mathr.ch/entry`, fill in `tension` + `context` + `contact`, submit. Should see a `📨 New /entry submission [EN]` message in `#mathr-dev`.

### CORS note

The `Respond OK` node sets `Access-Control-Allow-Origin: https://mathr.ch`. The form uses `application/x-www-form-urlencoded` content type, which is a "simple" CORS request (no preflight). If you later switch to `application/json` body, you'll need to handle OPTIONS preflight separately in n8n.

### Anti-spam strategy (Phase 0)

- **Honeypot field** `website` in the HTML form, hidden via off-screen CSS. Bots fill it; workflow drops those silently.
- **No CAPTCHA yet**. Phase 0 traffic is ~0; add Cloudflare Turnstile only if spam volume becomes real.
- **No rate limiting** at n8n level yet. Cloudflare in front of `n8n.mathr.ch` will absorb obvious DoS.

---

## Security TODO (separate from intake)

**n8n admin UI is currently exposed at `https://n8n.mathr.ch`** (the Cloudflare Tunnel proxies everything, not just `/webhook/*`). Anyone hitting the bare hostname sees the login page. Mitigations to consider:

1. **Cloudflare Access policy** on `n8n.mathr.ch` except `/webhook/*` paths — only admin emails get through to the UI, webhooks remain public. This is the cleanest option.
2. **Split hostnames** — keep `n8n.mathr.ch` for webhooks, move admin to `n8n-admin.mathr.ch` (or a non-public hostname) with Cloudflare Access. Requires reconfiguring the tunnel ingress.
3. **n8n built-in basic auth** — `N8N_BASIC_AUTH_ACTIVE=true` + strong credentials. Weakest of the three, but better than nothing.

Pick one before sharing `mathr.ch/entry` with anyone external.
