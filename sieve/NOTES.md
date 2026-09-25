# Notes

## GitHub webhook secret

The sieve loop is driven by a GitHub push webhook. GitHub signs each delivery with a shared secret, and Kargo checks the signature before refreshing the Warehouse.

The secret value is never stored in this repo.

### Where the secret lives

| Location | What | How it got there |
|----------|------|------------------|
| Kargo instance `marty-test`, namespace `sieve` | Secret `github-webhook-secret`, key `secret`, label `kargo.akuity.io/cred-type: generic` | `akuity kargo apply` from a local shell |
| GitHub `martysohio/kargo-computer` → Settings → Webhooks | Hook `685666642` (push events, JSON), signing secret | `gh api repos/martysohio/kargo-computer/hooks` |
| Local machine | `~/.kargo-sieve-webhook-secret` (mode 600) | `openssl rand -hex 32` at setup time |

The ProjectConfig in `kargo/projectconfig.yaml` refers to the Secret only by name (`webhookReceivers[].github.secretRef.name`).

### The receiver URL depends on the secret

Kargo builds the receiver URL from the receiver's configuration, secret included. It publishes the URL in the ProjectConfig's `status.webhookReceivers`. The URL is not recorded here: this repo is public, and the hard-to-guess path is part of what protects the endpoint. To look it up, check either:

- the ProjectConfig `sieve` status, in the Kargo UI or via MCP, or
- the GitHub hook's config: `gh api repos/martysohio/kargo-computer/hooks/685666642 --jq .config.url`

If the secret changes, the URL changes too, and the GitHub hook must be updated with both the new URL and the new secret.

### Rotating the secret

1. Generate a new secret and update the Kargo Secret:
   ```
   umask 077; openssl rand -hex 32 > ~/.kargo-sieve-webhook-secret
   akuity kargo apply --organization-id 7nfpt3kqvd0vclx0 --name marty-test -f - <<EOF
   apiVersion: v1
   kind: Secret
   metadata:
     name: github-webhook-secret
     namespace: sieve
     labels:
       kargo.akuity.io/cred-type: generic
   stringData:
     secret: $(cat ~/.kargo-sieve-webhook-secret)
   EOF
   ```
2. Read the new receiver URL from the ProjectConfig's `status.webhookReceivers`, in the Kargo UI or via MCP.
3. Update the GitHub hook:
   ```
   gh api -X PATCH repos/martysohio/kargo-computer/hooks/685666642 \
     -f 'config[url]=<new receiver URL>' -f 'config[content_type]=json' \
     -f "config[secret]=$(cat ~/.kargo-sieve-webhook-secret)"
   ```
4. Check that GitHub's redelivery or next push gets HTTP 200. Recent deliveries are listed with:
   ```
   gh api repos/martysohio/kargo-computer/hooks/685666642/deliveries --jq '.[] | {event, status_code, delivered_at}'
   ```

### Failure behaviour

- **Wrong or missing secret:** Kargo rejects the delivery, and GitHub shows a non-200 response under Recent Deliveries. The loop doesn't break, but each step then waits for the Warehouse's 15m backstop poll.
- **Deleting the local file** `~/.kargo-sieve-webhook-secret` is safe. It is only needed to update one side without rotating. Rotating (above) generates a new one.

## Git push credential

Promotions push to `main` using the shared Kargo git credential `martysohiogit`. It is defined at the instance's shared-credentials level, not in the `sieve` project. Kargo matches it by `repoURL`, which must be `https://github.com/martysohio/kargo-computer.git` (or a regex matching it).
