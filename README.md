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

Every key must be present, because `yaml-update` only updates keys that already exist:

```yaml
"n": 10          # quoted: YAML 1.1 reads a bare n as false
i: 2             # next number to process
done: false
primes: "[]"     # JSON array, stored as a string
lp: "[]"         # least prime factor of 0..n; padded with zeros automatically
```

### Starting a new run

Kargo commits to `main` on every step, so always pull first. Wait until the current run has finished (the last commit starts with `HALT:`) before starting another.

```sh
git pull --rebase origin main
printf '"n": 50\ni: 2\ndone: false\nprimes: "[]"\nlp: "[]"\n' > state.yaml
git add state.yaml
git commit -m "sieve: start run for n=50"
git push origin main
```

The push fires the GitHub webhook. Kargo turns the commit into Freight, auto-promotes it, and the loop runs by itself, about 8 seconds per step, until the `HALT:` commit. Follow along with:

```sh
git fetch -q && git log --oneline origin/main | head
```

The Stage `sieve` in the Kargo UI shows the same thing.

Only commits that change `state.yaml` start a promotion (the Warehouse sets `includePaths: [state.yaml]`). Commits touching other files, and empty commits, do not.

### Restarting a stalled loop

The loop stops without finishing (the last commit is a `sieve: i=...` step, not `HALT:`) in two cases:

- **A promotion failed.** Kargo won't auto-promote the same Freight again. Fix the cause (the Promotion's message in the Kargo UI says what failed), then either:
  - re-promote the newest Freight to Stage `sieve` from the Kargo UI, or
  - commit any change to `state.yaml`, for example a comment edit. The step reads `i` from the file, so the run carries on where it stopped:
    ```sh
    git pull --rebase origin main
    echo "# restart $(date -u +%FT%TZ)" >> state.yaml
    git commit -am "sieve: restart loop"
    git push origin main
    ```
- **A webhook delivery was missed.** Nothing needs doing: the Warehouse's 15m backstop poll picks up the commit. To go faster, use either option above.

To abandon a run and start over, follow [Starting a new run](#starting-a-new-run).

### Manifests (`kargo/`)

| file | resource |
|------|----------|
| `project.yaml` | Project `sieve` |
| `projectconfig.yaml` | ProjectConfig: auto-promotion for Stage `sieve`, GitHub webhook receiver |
| `warehouse.yaml` | Warehouse `sieve-state` |
| `stage.yaml` | Stage `sieve` |
| `tasks/git-checkout.yaml` | PromotionTask `git-checkout` |
| `tasks/euler-sieve-step.yaml` | PromotionTask `euler-sieve-step` |
| `tasks/git-commit-push.yaml` | PromotionTask `git-commit-push` |

Pushes use the shared git credential `martysohiogit` on the Kargo instance.

### Webhook

`projectconfig.yaml` defines a GitHub webhook receiver named `github`. It is backed by the Secret `github-webhook-secret` in namespace `sieve`: key `secret`, label `kargo.akuity.io/cred-type: generic`. The Secret is not kept in git. The GitHub repo has a push webhook (JSON, signed with the same secret) pointing at the receiver URL in the ProjectConfig's `status.webhookReceivers`. That URL changes if the secret changes.

See [NOTES.md](NOTES.md) for where the secret lives, how to rotate it, and the git push credential.
