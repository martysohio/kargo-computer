# kargo-computer
a weird computer using promotions

## Euler's sieve (Kargo project `sieve`)

Finds every prime `<= n` with Euler's (linear) sieve. Each Kargo promotion runs one step of the algorithm:

1. Warehouse `sieve-state` watches `state.yaml` on `main`. Each new commit becomes Freight.
2. Stage `sieve` auto-promotes the new Freight. Its promotion template chains three reusable PromotionTasks:
   - `git-checkout` clones `main`.
   - `euler-sieve-step` reads `state.yaml` with `yaml-parse`, processes the number `i`, and rewrites the file with `file-write`.
   - `git-commit-push` commits and pushes the result.
3. The push is a new commit, so the Warehouse produces new Freight and the loop runs again.
4. When `i > n` the step changes nothing, so nothing is committed and the loop stops with `done: true`.

`state.yaml` fields:

| key      | meaning |
|----------|---------|
| `"n"`    | input: find primes up to n (the key is quoted because YAML 1.1 reads a bare `n` as `false`) |
| `i`      | next number to process (starts at 2) |
| `primes` | primes found so far |
| `lp`     | least prime factor of every index `0..n` (0 = not yet known) |
| `done`   | true once every number up to n has been processed |

At step `i`: if `lp[i] == 0`, then `i` is prime. Then, for each prime `p <= lp[i]` with `i*p <= n`, set `lp[i*p] = p`. Every composite is marked exactly once, by its least prime factor. The commit message of each step records what happened (for example `sieve: i=4 lp=2, marked 8`).

To run again, commit a new state such as `"n": 50` with `i`, `primes`, `lp` and `done` removed.

Manifests are in [`kargo/sieve.yaml`](kargo/sieve.yaml).
