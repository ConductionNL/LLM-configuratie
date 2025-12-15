# Changelog

All notable changes to this repository will be documented in this file.

## 2025-12-12

- Initial GitOps setup for CPU LLMs
  - Helm chart `llama-cpp` (Deployment, Service, initContainer model download)
  - Values for `phi2`, `dolphin-phi2`, `mistral7b`
  - Argo CD Applications for three models
- Model download and auth
  - Init download via curl with optional `HUGGING_FACE_HUB_TOKEN` secret (`hf-token`)
  - Dolphin: switched to Mistral 7B GGUF mirror (`TheBloke/dolphin-2.6-mistral-7B-GGUF`) to avoid 401s on gated/original repos
- Image pull reliability
  - Fixed ImagePullBackOff by switching to Docker Hub image `ggerganov/llama.cpp:server`
- Scheduling and rollouts
  - Node scheduling via `nodeSelector: role=llm` and toleration `llm-only=true:NoSchedule`
  - Default rolling update: `maxUnavailable: 0`, `maxSurge: 1`
  - For dolphin and mistral values: `maxUnavailable: 1`, `maxSurge: 0` to avoid surge on single LLM node
  - Increased `progressDeadlineSeconds` to 1800 to accommodate large model downloads
- Chart defaults update
  - `infra/apps/llm/charts/llama-cpp/values.yaml`: added default `nodeSelector: role=llm`, `tolerations` for `llm-only`, and anti-affinity example to co-locate cleanly on LLM nodes
  - Image repository updated per environment to a reachable registry; tag remains `server`
- Documentation
  - Root `README.md` with overview and quick start
  - `docs/README.md` with in-cluster/external access, operations alignment, secrets rotation, and generic DB primary/secondary guidance
- Argo CD Applications
  - Kept original `repoURL` (non-.git) to avoid fetch timeouts; no functional change to sync logic

### Bugfixes
- ImagePullBackOff on `ghcr.io/ggerganov/llama.cpp:server` (tag not found) → use `ggerganov/llama.cpp:server` (Docker Hub)
- Dolphin 401 during model download from gated repo → switch to public GGUF mirror or use valid HF token
- ProgressDeadlineExceeded during long model downloads → increase `progressDeadlineSeconds`


