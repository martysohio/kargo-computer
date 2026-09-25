# kargo-computer
a weird computer using promotions

## Euler's sieve (Kargo project `sieve`)

Finds every prime `<= n` with Euler's (linear) sieve. Kargo runs it as a self-clocking state machine:

| part  | what it is |
|-------|------------|
| RAM   | `state.yaml` on `main` |
| Clock | a GitHub push webhook refreshes Warehouse `sieve-state`, which turns each commit to `state.yaml` into Freight (a 15m poll is the backstop) |
| CPU   | Stage `sieve` runs one sieve iteration per auto-promotion |
| Halt  | once `i > n` the step changes nothing, so nothing is pushed and the loop stops |

Each promotion chains three reusable PromotionTasks:

- `git-checkout` clones `main`.
- `euler-sieve-step` runs fetch (`yaml-parse`), decode, execute and cross-off (`compose-output`), then write-back (`yaml-update`).
- `git-commit-push` commits and pushes. The commit message records the step, e.g. `sieve: i=4 lp=2, marked 8`. The last one reads `HALT: primes <= 10: [2, 3, 5, 7]`.

At step `i`: if `lp[i] == 0`, then `i` is prime. Then, for every prime `p <= lp[i]` with `i*p <= n`, set `lp[i*p] = p`. Every composite is crossed off exactly once, by its least prime factor.

### state.yaml

Every key must be present, because `yaml-update` only updates keys that already exist. To start a new run, commit:

```yaml
"n": 10          # quoted: YAML 1.1 reads a bare n as false
i: 2
done: false
primes: "[]"     # JSON array, stored as a string
lp: "[]"         # least prime factor of 0..n; padded with zeros automatically
```

### Manifests (`kargo/`)

| file | resource |
|------|----------|
| `project.yaml` | Project `sieve` |
| `projectconfig.yaml` | ProjectConfig: auto-promotion for Stage `sieve` |
| `warehouse.yaml` | Warehouse `sieve-state` |
| `stage.yaml` | Stage `sieve` |
| `tasks/git-checkout.yaml` | PromotionTask `git-checkout` |
| `tasks/euler-sieve-step.yaml` | PromotionTask `euler-sieve-step` |
| `tasks/git-commit-push.yaml` | PromotionTask `git-commit-push` |

Pushes use the shared git credential `martysohiogit` on the Kargo instance.

### Webhook

`projectconfig.yaml` defines a GitHub webhook receiver named `github`. It is backed by the Secret `github-webhook-secret` in namespace `sieve`: key `secret`, label `kargo.akuity.io/cred-type: generic`. The Secret is not kept in git. The GitHub repo has a push webhook (JSON, signed with the same secret) pointing at the receiver URL in the ProjectConfig's `status.webhookReceivers`. That URL changes if the secret changes.
