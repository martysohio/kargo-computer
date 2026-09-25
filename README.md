# kargo-computer
a weird computer using promotions

## Examples

Each example lives in its own directory, with its state file and its Kargo manifests.

| directory | Kargo project | what it computes |
|-----------|---------------|------------------|
| [`sieve/`](sieve/) | `sieve` | all primes `<= n` with Euler's (linear) sieve, one step per promotion |

The GitHub push webhook on this repo belongs to the `sieve` project's receiver. A new example that is also driven by pushes needs its own webhook receiver and repo webhook. Its Warehouse's `includePaths` should cover only that example's directory, so examples don't trigger each other.
