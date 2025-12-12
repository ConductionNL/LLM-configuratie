# LLM-configuratie (CPU, GitOps met Argo CD + Helm)

Deze repo levert een minimale Helm chart om CPU-gebaseerde LLM-servers (llama.cpp) op Kubernetes uit te rollen, beheerd via Argo CD. Modellen worden bij start gedownload (GGUF) vanuit Hugging Face met een initContainer. Standaard geldt geen specifieke node-binding; optionele scheduling kun je zelf toevoegen.

## Inhoud
- Overzicht en architectuur
- Vereisten
- Deployen (GitOps/Argo CD)
- Secret voor Hugging Face
- Testen en gebruiken
- Waarden aanpassen (per model)
- Troubleshooting
- Nieuw model toevoegen

## Overzicht en architectuur
- Chart: `infra/apps/llm/charts/llama-cpp`
- Values (modellen): `infra/apps/llm/values/*.yaml`
- Argo CD Applications: `infra/apps/llm/argocd/*.yaml`
- Namespace: `llm`
- Model-download: initContainer met `curl` naar `/models` (emptyDir), server gebruikt `-m /models/<bestandsnaam>.gguf`

## Vereisten
- Kubernetes cluster met 1 node voor LLM workloads:
  - label: `llm-node=true`
  - taint: `llm-only=true:NoSchedule`
- Namespace `llm`
- Argo CD geïnstalleerd in namespace `argocd`
- Secret `hf-token` in namespace `llm` met key `HUGGING_FACE_HUB_TOKEN`

## Deployen (GitOps/Argo CD)
1) Commit & push de repo naar `main` (reeds gedaan in init).  
2) Apply Argo CD Applications:

```bash
kubectl apply -n argocd -f infra/apps/llm/argocd/
kubectl -n argocd get applications
```

Argo CD synct nu:
- `llm-phi2` met `infra/apps/llm/values/phi2.yaml`
- `llm-dolphin-phi2` met `infra/apps/llm/values/dolphin-phi2.yaml`
- `llm-mistral7b` met `infra/apps/llm/values/mistral7b.yaml`

## Secret voor Hugging Face
Maak de secret in namespace `llm` (geen secrets in Git):

```bash
kubectl -n llm create secret generic hf-token \
  --from-literal=HUGGING_FACE_HUB_TOKEN=<JOUW_HF_TOKEN>
```

De initContainer en server lezen deze variabele (optioneel, voor private of rate-limited downloads).

## Testen en gebruiken
Services (voorbeeld):  
`llm-phi2-llama-cpp`, `llm-dolphin-phi2-llama-cpp`, `llm-mistral7b-llama-cpp`

Port-forward (kies een vrije lokale poort, bijv. 18080):

```bash
kubectl -n llm port-forward service/llm-phi2-llama-cpp 18080:8080
```

Sanity check:

```bash
curl -sS http://localhost:18080
curl -sS -X POST http://localhost:18080/completion \
  -H "Content-Type: application/json" \
  -d '{"prompt":"Hello","n_predict":16}'
```

OpenAI-achtig endpoint (indien door image ondersteund):

```bash
curl -sS -X POST http://localhost:18080/v1/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"phi2","prompt":"Hello","max_tokens":16}'
```

### In-cluster toegang (zonder port-forward)
Indien je workload binnen het cluster draait, gebruik dan de ClusterIP Service via DNS. Port-forward is alleen nodig voor lokale toegang vanaf je laptop.

- Directe DNS van de service: `llm-phi2-llama-cpp.llm.svc.cluster.local:8080`
- Binnen dezelfde namespace (`llm`) volstaat `llm-phi2-llama-cpp:8080`

Voorbeeld (ad-hoc curl pod):

```bash
kubectl -n llm run curl --rm -it --image=curlimages/curl --restart=Never -- \
  sh -lc 'curl -sS http://llm-phi2-llama-cpp:8080/completion -H "Content-Type: application/json" -d "{\"prompt\":\"Hello\",\"n_predict\":16}"'
```

Voorbeeld (consumer Deployment die de LLM-service aanroept):

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-consumer
  namespace: llm
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llm-consumer
  template:
    metadata:
      labels:
        app: llm-consumer
    spec:
      containers:
        - name: app
          image: curlimages/curl:8.10.1
          command: ["sh","-lc"]
          args:
            - >
              while true; do
                echo "Calling LLM...";
                curl -sS http://llm-phi2-llama-cpp.llm.svc.cluster.local:8080/completion
                echo; sleep 30;
              done
```

### Externe toegang (optioneel)
Standaard zijn Services `ClusterIP` en dus alleen in-cluster bereikbaar. Voor externe toegang kies één van de patronen:

- LoadBalancer (L4):
  - Zet per model in de values-file:
    ```yaml
    service:
      type: LoadBalancer
      port: 8080
    ```
  - Let op netwerkbeperkingen (firewall/IP-allowlist) bij je cloud LB.

- Ingress (L7, aanbevolen met TLS + auth):
  - Laat de service `ClusterIP` en maak een Ingress via je Ingress Controller (bijv. NGINX):
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: llm-phi2
      namespace: llm
      annotations:
        nginx.ingress.kubernetes.io/proxy-body-size: "0"
        # voeg hier auth/tls/ratelimiting annotaties toe
    spec:
      ingressClassName: nginx
      tls:
        - hosts: ["phi2.example.com"]
          secretName: phi2-tls
      rules:
        - host: phi2.example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: llm-phi2-llama-cpp
                    port:
                      number: 8080
    ```
  - Bescherm altijd met TLS en (bij voorkeur) authenticatie of netwerkbeperkingen.

## Operationele alignment (hoe dit samenhangt)
- GitOps bron van waarheid: deze repo. Argo CD leest `infra/apps/llm/argocd/*.yaml` en rendert Helm chart `infra/apps/llm/charts/llama-cpp` met per-model `values/*.yaml`.
- Scheduling: standaard generiek (geen node-binding). Indien gewenst, voeg `nodeSelector` en `tolerations` toe per values-file.
- Rollouts: default RollingUpdate. Voor single-node omgevingen kun je:
  - Minimale downtime (extra pod): `maxUnavailable: 0`, `maxSurge: 1`
  - Zero-surge (sequentieel): `maxUnavailable: 1`, `maxSurge: 0`
- Langdurige init (model-download) kan de standaard progress-deadline overschrijden; deze chart zet een ruime `progressDeadlineSeconds` in om dat te voorkomen.

## Secrets en rotatie (incl. wachtwoorden)
Deze chart gebruikt géén database. Wel is er een secret voor Hugging Face optioneel:

```bash
kubectl -n llm create secret generic hf-token \
  --from-literal=HUGGING_FACE_HUB_TOKEN=<JOUW_HF_TOKEN> \
  --dry-run=client -o yaml | kubectl apply -f -
```

Algemene richtlijnen voor secret-rotatie (ook toepasbaar op DB-wachtwoorden voor aanpalende apps):
1) Maak/patch de Secret met de nieuwe waarde (zoals hierboven) in de juiste namespace.
2) Forceer een rollout van de workloads die de Secret consumeren, zodat env vars opnieuw geladen worden. Voor deze chart:
```bash
kubectl -n llm rollout restart deploy/llm-phi2-llama-cpp
kubectl -n llm rollout restart deploy/llm-dolphin-phi2-llama-cpp
kubectl -n llm rollout restart deploy/llm-mistral7b-llama-cpp
```
3) Controleer dat pods de nieuwe config hebben en functioneren.

Pattern voor “rollout bij secret-wijziging” zonder handmatig restart:
- Voeg een annotatie toe in je values, en “bump” die bij wijziging zodat Argo CD een nieuwe PodTemplate afleidt (en dus een rollout triggert). Voorbeeld in een values-file:
```yaml
podAnnotations:
  config/refresh: "2025-12-12-12:00"  # verander timestamp/string bij rotatie
```

## Primair/secundair database (algemene richtlijn)
Niet van toepassing op deze chart (llama.cpp heeft geen DB nodig), maar voor applicaties die wél een DB gebruiken:
- Gebruik voor writes een “primary/write” endpoint (VIP/Service), en voor reads een “read” endpoint indien de app dat ondersteunt.
- Bewaar DB-credentials in een Kubernetes Secret (per omgeving/namespace).
- Rotatie zonder downtime (gangbaar patroon):
  1) Voeg nieuw wachtwoord toe aan de DB (oude blijft tijdelijk geldig).
  2) Update de Secret in Kubernetes met het nieuwe wachtwoord.
  3) Forceer rollout van de betrokken Deployments/StatefulSets (zoals hierboven).
  4) Verifieer dat de app met het nieuwe wachtwoord werkt.
  5) Verwijder pas daarna het oude wachtwoord uit de DB.
- Failover primary→secondary gebeurt idealiter in de DB-laag of via de connectiestring (bijv. meerdere hosts). Apps wijzen naar het write-endpoint; bij failover blijft de connectie-URL gelijk.

## Waarden aanpassen (per model)
Voor elk model is er een values-file met o.a.:

```yaml
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
model:
  enabled: true
  repoId: "TheBloke/phi-2-GGUF"
  filename: "phi-2.Q4_K_M.gguf"
#  downloadUrl: ""  # optioneel: zet direct-URL om 403/404 te omzeilen
extraArgs:
  - "-m"
  - "/models/phi-2.Q4_K_M.gguf"
  - "-c"
  - "4096"
  - "-t"
  - "4"
  - "--port"
  - "8080"
  - "--host"
  - "0.0.0.0"
```

Let op:
- `repoId` + `filename` gebruikt een standaard `https://huggingface.co/<repoId>/resolve/main/<filename>` URL.
- Gebruik `downloadUrl` om een exacte download-URL te forceren (handig bij 403/404 of LFS-redirects).
- Pas `-t` aan op het aantal CPU-cores dat je wil gebruiken.

## Troubleshooting
### Port-forward faalt (poort bezet)
Fout: `address already in use`. Kies een andere lokale poort, bv. `28080:8080`, of laat Kubernetes kiezen: `:8080`.

### Pod scheduled op verkeerde node
Check label/taint:
```bash
kubectl get nodes -L llm-node
kubectl -n llm get pods -o wide
```
Node zonder label `llm-node=true` accepteert deze pods niet.

### Init CrashLoopBackOff (download-model)
1) Bekijk logs van initContainer:
```bash
kubectl -n llm logs <POD_NAME> -c download-model --tail=200
kubectl -n llm describe pod <POD_NAME>
```
2) Controleer:
- Bestaat de secret `hf-token` met key `HUGGING_FACE_HUB_TOKEN`?
- Klopt `repoId` en `filename`?
- Probeer een directe URL in values: `model.downloadUrl: "https://huggingface.co/<repoId>/resolve/main/<filename>"`.
- Rate limiting (429/403): voeg token toe of probeer later opnieuw.

### Service bereikbaar, maar fouten bij /completion
- Controleer dat `-m /models/<bestandsnaam>.gguf` overeenkomt met het gedownloade bestand.
- Check container-logs:
```bash
kubectl -n llm logs <POD_NAME> -c server --tail=200
```

## Nieuw model toevoegen
1) Kopieer een values-file in `infra/apps/llm/values/` en pas `repoId`, `filename`, `resources`, `extraArgs` aan.
2) Maak een Argo CD Application (kopie van bestaande in `infra/apps/llm/argocd/`) en verwijs naar je nieuwe values-file in `helm.valueFiles`.
3) Commit en laat Argo CD syncen.


