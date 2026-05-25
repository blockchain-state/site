# ops

Operational artifacts for Math.R infrastructure. Not deployed with the site.

## n8n-github-to-kchat.json

Workflow: **GitHub → n8n → kChat #mathr-dev**

### Setup

1. **Import in n8n**: Workflows → Import from File → pick this JSON.
2. **Set env var on the n8n host**:
   ```
   KCHAT_WEBHOOK_URL=https://mathr.kchat.infomaniak.com/hooks/<token>
   ```
   Restart n8n so it picks up the var.
3. **Activate** the workflow (toggle top-right). The webhook node will show its production URL:
   ```
   https://<your-n8n-host>/webhook/github-mathr
   ```
4. **Register in GitHub** repo `01ehex/mathr.ch`:
   - Settings → Webhooks → Add webhook
   - Payload URL: `https://<your-n8n-host>/webhook/github-mathr`
   - Content type: `application/json`
   - Secret: (optional but recommended — see below)
   - Events: **Just the push event** + select **Pull requests** in "Let me select individual events"
   - Active: ✓

### Verifying

GitHub sends a `ping` event on registration. You should see in #mathr-dev:

> 🔌 webhook connected: **01ehex/mathr.ch**

Then push a commit on `main` to confirm the push path.

### Optional: secret verification

To verify GitHub's signature, add a node between the webhook and Switch:

- Set the same secret in GitHub webhook settings.
- In the Code node, verify `$json.headers['x-hub-signature-256']` against HMAC-SHA256 of the raw body with that secret.

For Phase 0 this is fine without — your n8n is presumably not publicly indexed and the webhook path is unguessable.

### Channel routing (later)

When you split per-track (vayu, edg, mars, bs), extend the Switch node to inspect `$json.body.repository.name` and send to different kChat webhook URLs per channel.
