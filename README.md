# Spring PetClinic — GenAI Chatbot Integration (PETC-708)

## Overview

Integrated a production-grade GenAI chatbot into the Spring PetClinic microservices application as part of DMI Cohort 2. The chatbot uses Spring AI with Azure OpenAI (GPT-4o-mini) for conversation and text-embedding-ada-002 for semantic vector search over veterinary data. Deployed on Azure Kubernetes Service (AKS) via Helm and Azure Pipelines.

**Live deployment:** https://lnkd.in/drZczvfg

---

## Architecture

```
User Browser
    │
    ▼  HTTP
Azure Load Balancer (AKS Service: LoadBalancer)
    │
    ▼
Spring Boot — api-gateway (port 8080)
    │  Routes /chatbot/** to chatbot-service
    ▼
Spring Boot — chatbot-service (port 8084)
    ├── POST /chatbot/chat         → GPT-4o-mini (Azure OpenAI)
    ├── POST /chatbot/embed        → text-embedding-ada-002 (Azure OpenAI)
    └── GET  /chatbot/search       → Vector Store (in-memory or Azure AI Search)
         │
         │  Azure OpenAI API calls
         ▼
Azure OpenAI Service
    ├── Deployment: gpt-4o-mini        (chat completions)
    └── Deployment: text-embedding-ada-002  (embeddings)
```

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Application | Spring Boot 3.x, Spring AI 1.0 |
| AI | Azure OpenAI (GPT-4o-mini, text-embedding-ada-002) |
| Container | Docker (multi-stage build, Azure Container Registry) |
| Orchestration | Kubernetes on AKS |
| Packaging | Helm |
| CI/CD | Azure Pipelines (YAML, multi-stage) |
| Monitoring | Grafana + Prometheus |
| IaC | Terraform (AKS cluster) |

---

## Key Engineering Challenges Solved

### 1. ISP Blocking Docker Hub on Pull

**Problem:** Docker Hub image pulls failed silently behind a corporate ISP with hub.docker.com blocked.

**Solution:** Import all base images into Azure Container Registry (ACR) using `az acr import`:

```bash
az acr import \
  --name dmiregistry \
  --source docker.io/library/openjdk:17-alpine \
  --image openjdk:17-alpine

az acr import \
  --name dmiregistry \
  --source docker.io/library/maven:3.9-eclipse-temurin-17 \
  --image maven:3.9-eclipse-temurin-17
```

Dockerfiles updated to reference `dmiregistry.azurecr.io/` instead of `docker.io/`.

---

### 2. Reactive Threading Crash (WebFlux + Blocking DB Call)

**Problem:** Spring AI's embedding pipeline used a blocking JDBC call inside a WebFlux reactive chain, causing:
```
java.lang.IllegalStateException: block()/blockFirst()/blockLast() are blocking
```

**Root cause:** Calling `.block()` on a reactive thread (Netty event loop) is prohibited — it would deadlock the event loop.

**Solution:** Wrap the blocking operation in a bounded elastic scheduler:

```java
// WRONG — blocks the reactive thread
Mono<List<Document>> result = Mono.fromCallable(() -> vectorStore.search(query))
    .block();

// CORRECT — offloads blocking call to a separate thread pool
Mono<List<Document>> result = Mono.fromCallable(() -> vectorStore.search(query))
    .subscribeOn(Schedulers.boundedElastic());
```

---

### 3. Exposed API Key in Git History

**Problem:** Azure OpenAI API key was committed in `application.properties` and pushed to GitHub.

**Solution:** Removed the credential from all commits using `git filter-repo`:

```bash
pip install git-filter-repo

git filter-repo \
  --path src/main/resources/application.properties \
  --invert-paths

# Force push cleaned history
git push origin --force --all
git push origin --force --tags
```

The API key was rotated immediately after discovery. Credential was moved to Kubernetes Secret and injected as an environment variable.

---

### 4. Empty Vector Store (Embedding Model Mismatch)

**Problem:** Semantic search returned no results even when matching documents existed.

**Root cause:** The embedding model used to index documents (`text-embedding-ada-002`, 1536 dimensions) was different from the model used at query time (a different model with different dimensions). Vector dimensions did not match — cosine similarity was undefined.

**Solution:** Explicitly configure the same embedding model for both indexing and querying in `application.yaml`:

```yaml
spring:
  ai:
    azure:
      openai:
        embedding:
          options:
            model: text-embedding-ada-002
```

All documents were re-indexed after the configuration fix.

---

## Local Development Setup

### Prerequisites

```bash
# Java 17
java -version

# Maven
mvn -version

# kubectl connected to AKS
kubectl cluster-info
```

### Clone and configure

```bash
git clone https://github.com/pravinmishraaws/spring-petclinic-microservices.git
cd spring-petclinic-microservices
```

Create `chatbot-service/src/main/resources/application-local.properties`:

```properties
spring.ai.azure.openai.endpoint=https://YOUR_ENDPOINT.openai.azure.com/
spring.ai.azure.openai.api-key=${AZURE_OPENAI_API_KEY}
spring.ai.azure.openai.chat.options.deployment-name=gpt-4o-mini
spring.ai.azure.openai.embedding.options.model=text-embedding-ada-002
```

> **Never commit API keys.** Set `AZURE_OPENAI_API_KEY` as an environment variable.

### Build and run locally

```bash
mvn clean package -DskipTests
java -jar chatbot-service/target/chatbot-service.jar --spring.profiles.active=local
```

---

## Kubernetes Deployment (AKS)

### 1. Create Kubernetes Secret for API key

```bash
kubectl create secret generic azure-openai-secret \
  --from-literal=api-key=$AZURE_OPENAI_API_KEY \
  --namespace petclinic
```

### 2. Deploy with Helm

```bash
helm upgrade --install petclinic ./helm/petclinic \
  --namespace petclinic \
  --create-namespace \
  --set chatbot.image.tag=$(Build.BuildNumber) \
  --set chatbot.azure.openai.endpoint=$AZURE_OPENAI_ENDPOINT
```

### 3. Verify the chatbot service

```bash
kubectl get pods -n petclinic
kubectl logs -n petclinic deployment/chatbot-service

# Port-forward to test locally
kubectl port-forward svc/chatbot-service 8084:8084 -n petclinic
curl -X POST http://localhost:8084/chatbot/chat \
  -H "Content-Type: application/json" \
  -d '{"message": "What vaccines does a dog need?"}'
```

---

## Azure Pipeline (CI/CD)

See `azure-pipelines.yml` for the full multi-stage pipeline:

1. **Build** — `mvn clean package`, Docker multi-stage build, push to ACR
2. **Test** — Unit tests with JUnit, publish results
3. **Deploy** — Helm upgrade on AKS, `kubectl rollout status`

---

## Monitoring (Grafana + Prometheus)

Grafana dashboard tracks:
- ChatBot request rate (requests/min)
- Azure OpenAI API latency (p50, p95, p99)
- Vector store query duration
- Error rate per endpoint

Access: `http://<GRAFANA_SERVICE_IP>:3000`

---

## ⚠️ Key Corrections from Development

| Issue | What happened | Fix |
|-------|---------------|-----|
| API key exposed | Committed to git, pushed to GitHub | `git filter-repo` to scrub history, rotate key, use K8s Secret |
| Reactive threading crash | `.block()` on Netty thread | `Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())` |
| Docker Hub blocked | ISP blocked hub.docker.com | `az acr import` all base images into ACR |
| Empty vector search | Embedding model dimension mismatch | Set identical model for index + query in `application.yaml` |

---

## Key Learnings

- Spring AI's `VectorStore` and `ChatClient` are reactive-compatible but the underlying JDBC/embedding operations may be blocking — always wrap with `Schedulers.boundedElastic()` in a WebFlux context
- `git filter-repo` is the current standard for scrubbing secrets from git history (`git filter-branch` is deprecated and unsafe)
- Azure OpenAI embedding dimensions must match exactly between index time and query time — a mismatch produces silent failures (no results, no error)
- Kubernetes Secrets are the correct pattern for API keys in AKS — not environment variables in Helm values files, not ConfigMaps
- `az acr import` bypasses Docker Hub rate limits and ISP blocks by pulling images at the Azure network level and storing them in your private registry
