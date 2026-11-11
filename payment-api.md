# Runbook: payment-processor architecture mismatch

A `payment-processor` release built for the wrong CPU architecture fails on start and crash-loops. `payment-api` and `payment-reconciliation` fail after it because they depend on it. Roll back, then rebuild the image for every node architecture.

Scenario: `./generate payments-arch-mismatch`

## Symptoms

| Service | Alert | Meaning |
| --- | --- | --- |
| payment-processor | Deployment rollout stalled, new ReplicaSet 0/6 ready | New pods never start |
| payment-processor | Pod CrashLoopBackOff, exit 255 | The binary can't execute on the node |
| payment-processor | ALB targets unhealthy | Too few healthy pods behind the load balancer |
| payment-api | p99 latency over SLO, 5xx 502/503 | Upstream processor unavailable |
| payment-api | Pod CrashLoopBackOff, exit 1 | Startup check can't reach payment-processor |
| payment-api | Synthetics: checkout fails at Submit payment | Customer impact |
| payment-reconciliation | recon-worker CrashLoopBackOff, exit 1 | Can't fetch authorisations from payment-processor |
| payment-reconciliation | Settlement batch has unmatched transactions | Authorisations missing from payment-processor |

Ignore the FASTCHECKOUT flag rollout and the `/var/log` disk alert. Neither can cause an exec format error.

## Diagnose

Run these in order. Each step is one workflow action.

| # | Workflow action | Inputs | Points to the root cause when |
| --- | --- | --- | --- |
| 1 | Find the first crashing service | `payment-processor`, `payment-api`, `payment-reconciliation` | `payment-processor` crash-loops first |
| 2 | Read last log line of crashing pod | `payment-processor` | `exec /app/payment-processor: exec format error` |
| 3 | Read container exit code | `payment-processor` | 255 on start (not 137 OOM, not a runtime panic) |
| 4 | Get recent deployments | `payment-processor` | v3.3.0 replaced v3.2.1 just before the first alert |
| 5 | Get image platforms | image `payment-processor:v3.3.0` | Only `linux/arm64` |
| 6 | Get node architecture | nodes running `payment-processor` | `amd64` |
| 7 | Get build runner for release | `payment-processor` v3.3.0 | Built on `ubuntu-24.04-arm` |

If steps 5 and 6 disagree, the image can't run on the nodes, and that is the root cause. The `payment-api` and `recon-worker` logs show `connection refused` to `payment-processor`, not an exec error, so they are downstream.

## Fix

| # | Workflow action | Inputs | Pass when |
| --- | --- | --- | --- |
| 1 | Roll back deployment | `payment-processor`, target v3.2.1 | 6 of 6 pods Running and Ready |
| 2 | Restart dependent pods | `payment-api`, `payment-reconciliation` | Crash loops stop |
| 3 | Check load balancer targets | `payment-processor-prod` | 6 of 6 healthy |
| 4 | Run synthetic check | `checkout-journey` | Passes from all 4 locations |
| 5 | Rerun settlement batch | batch T1 | Zero unmatched transactions |

## Before redeploying v3.3.0

- Rebuild as a multi-arch image: `linux/amd64,linux/arm64`.
- Rerun Diagnose step 5 and confirm it lists both platforms.
- Don't raise resource limits or toggle flags. Neither fixes an exec format error.

