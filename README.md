# LLM-configuratie

CPU-gebaseerde LLM-servers (llama.cpp) uitgerold op Kubernetes via GitOps (Argo CD + Helm). Modellen (GGUF) worden bij start gedownload vanaf Hugging Face via een initContainer. Zie uitgebreide handleiding in `docs/README.md`.

## Structuur
```
infra/
  apps/
    llm/
      charts/
        llama-cpp/          # Helm chart: Deployment + Service + init download
      values/
        phi2.yaml           # Phi-2 (2.7B)
        dolphin-phi2.yaml   # Dolphin (gebruikt Mistral 7B GGUF)
        mistral7b.yaml      # Mistral 7B (Instruct)
      argocd/
        llm-phi2.yaml
        llm-dolphin-phi2.yaml
        llm-mistral7b.yaml
docs/
  README.md                 # Volledige documentatie (gebruik, troubleshooting, extern)
```

## Vereisten (kort)
- Kubernetes cluster met namespace `llm`
- Argo CD (namespace `argocd`)
- Secret met Hugging Face token (optioneel, voor private/gated of rate-limited downloads):
  ```bash
  kubectl -n llm create secret generic hf-token \
    --from-literal=HUGGING_FACE_HUB_TOKEN=<YOUR_HF_TOKEN>
  ```

## Deploy (GitOps)
1) Argo CD Applications toepassen
```bash
kubectl apply -n argocd -f infra/apps/llm/argocd/
kubectl -n argocd get applications
```
2) Status controleren
```bash
kubectl -n llm get pods -o wide
kubectl -n llm get svc
```

## Snel testen
- Port-forward (lokaal):
```bash
kubectl -n llm port-forward service/llm-phi2-llama-cpp 18080:8080
curl -sS -X POST http://localhost:18080/completion \
  -H "Content-Type: application/json" \
  -d '{"prompt":"Hello","n_predict":16}'
```
- In-cluster (DNS):
```bash
kubectl -n llm run curl --rm -it --image=curlimages/curl --restart=Never -- \
  sh -lc 'curl -sS http://llm-phi2-llama-cpp.llm.svc.cluster.local:8080/completion \
  -H "Content-Type: application/json" -d "{\"prompt\":\"Hello\",\"n_predict\":16}"'
```

## Externe toegang
Standaard `ClusterIP` (alleen in-cluster). Voor extern:
- Service type `LoadBalancer`, of
- Ingress (aanbevolen) met TLS + auth.

Details en troubleshooting: zie `docs/README.md`.

