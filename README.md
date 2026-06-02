# CAPZ Prow Dashboard — Qwen / Dynamo Demo

This repo is the **demo variant** of the [Cluster API Provider Azure
prow dashboard](https://github.com/willie-yao/capz-prow-ai-dashboard).
Same engine ([willie-yao/prow-ai-dashboard](https://github.com/willie-yao/prow-ai-dashboard)),
same prow jobs, same prompt, same evidence — different model and
endpoint.

| | Prod | Demo (this repo) |
|---|---|---|
| Dashboard | https://willie-yao.github.io/capz-prow-ai-dashboard/ | https://willie-yao.github.io/capz-prow-ai-dashboard-qwen/ |
| Model | Claude Opus (GitHub Models) | Qwen3-235B-A22B-FP8 |
| Endpoint | `api.githubcopilot.com` | In-cluster Dynamo frontend (cluster DNS) |
| Runner | `ubuntu-latest` (GitHub-hosted) | `arc-qwen-runner` (self-hosted, in cluster) |
| Tool calling | Native | Native (Hermes parser via `--dyn-tool-call-parser hermes`) |
| Reasoning routing | n/a | `<think>` blocks split into `reasoning_content` |

## Why this repo exists

Phase L's universal AI path lets any OpenAI-compatible chat-completions
endpoint serve the same agentic evidence-gathering loop, so this
deployment validates two things at once:

1. **Engine portability** — same code, same prompts, swap the endpoint
   and model, get useful summaries on real CAPZ failures.
2. **Cluster-internal AI plumbing** (Phase K.3) — proves end-to-end
   that a private model served from your own cluster can drive the
   dashboard without any data leaving the cluster.

The prod dashboard stays on Claude (high signal) so this experimental
variant doesn't risk the trustworthy production signal.

## Setup

One-time, before the first deploy succeeds:

1. **Install actions-runner-controller (ARC)** in the demo AKS cluster
   with a runner pool labelled `arc-qwen-runner`. Runner pods need
   cluster DNS access to the Dynamo frontend service in the `default`
   namespace.

2. **Set repo variables**:
   ```bash
   gh variable set AI_ENDPOINT \
     -b "http://qwen3-235b-a22b-disagg-frontend.default.svc:8000/v1/chat/completions"
   gh variable set AI_MODEL -b "Qwen/Qwen3-235B-A22B-FP8"
   ```

3. **Set `AI_TOKEN` secret** to any non-empty string. Dynamo doesn't
   require an API key, but the engine asserts the secret is set.
   ```bash
   gh secret set AI_TOKEN -b "not-required-for-dynamo"
   ```

4. **Verify worker flags** on the trtllm pods (one-time, your teammate
   already configured these):
   - `--dyn-tool-call-parser hermes` (structured `tool_calls`)
   - `--dyn-reasoning-parser qwen3` (split `<think>` into `reasoning_content`)

## Local POC / debugging

To iterate on prompts or compare model responses without round-tripping
through GitHub Actions, the engine ships a single-failure CLI:

```bash
# Port-forward Dynamo to your laptop (one terminal):
kubectl port-forward -n default svc/qwen3-235b-a22b-disagg-frontend 8000:8000

# Build the spike (one-time):
cd ~/go/prow-ai-dashboard/backend
go build -o /tmp/spike ./cmd/ai-toolcall-spike

# Run against any prow build, Qwen via local port-forward:
AI_TOKEN="dummy" /tmp/spike \
  -endpoint "http://localhost:8000/v1/chat/completions" \
  -model "Qwen/Qwen3-235B-A22B-FP8" \
  -job "periodic-cluster-api-provider-azure-e2e-main" \
  -build "<build-id>" \
  -test "<test name>" \
  -failure "<failure message excerpt>" \
  -v

# Same prow build, Claude on GH Models for A/B:
AI_TOKEN="$GITHUB_TOKEN" /tmp/spike \
  -endpoint "https://api.githubcopilot.com/chat/completions" \
  -model "claude-opus-4.7-xhigh" \
  -job "..." -build "..." -test "..." -failure "..." -v
```

## What stays in lockstep with the prod CAPZ dashboard

The whole point of this repo is to be an **A/B variant**, not a fork.
Drift kills the comparison.

Keep identical to https://github.com/willie-yao/capz-prow-ai-dashboard :

- `prompts/system.md` (project-specific knowledge)
- `project.yaml`: everything except `id`, `branding`, and the AI
  endpoint/model (which come from repo variables anyway)

If the prod dashboard changes its prompt or evidence config, mirror
the change here within a few days so the comparison stays meaningful.
