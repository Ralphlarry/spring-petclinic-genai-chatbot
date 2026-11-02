# Spring PetClinic GenAI chatbot (PETC-708)

My part of the final group project on the DevOps Micro Internship (DMI Cohort 2, 2026). Group 9 deployed the Spring PetClinic microservices app to AKS. My Jira ticket, PETC-708, was getting the app's existing GenAI chat service working on Azure OpenAI.

The code is in the group's repo, [spring-petclinic-devops](https://github.com/dmi-petclinic-devops-team-group-9/spring-petclinic-devops). This repo is only my write-up of it. The group's live site was taken down after the internship to save cost.

## What I changed

My merged PRs are [#25](https://github.com/dmi-petclinic-devops-team-group-9/spring-petclinic-devops/pull/25) and [#40](https://github.com/dmi-petclinic-devops-team-group-9/spring-petclinic-devops/pull/40), both on `spring-petclinic-genai-service`.

**Azure OpenAI.** I swapped `spring-ai-starter-model-openai` for `spring-ai-starter-model-azure-openai` and set the chat deployment to `gpt-4o-mini`. The API key comes from an environment variable, backed by a Kubernetes Secret.

**Reactive threading crash.** The chat endpoint made a blocking call on a WebFlux thread. I moved that call onto a separate thread pool:

```java
return Mono.fromCallable(() ->
        this.chatClient.prompt().user(query).call().content()
    )
    .subscribeOn(Schedulers.boundedElastic());
```

**Empty vector search.** This came from an embedding model mismatch. The service loads `vectorstore.json` if that file exists, so it was using vectors it hadn't built itself. I removed the file, which makes the service rebuild the store at startup, and set the embedding deployment to `text-embedding-ada-002`.

**Docker Hub blocked.** My ISP blocks Docker Hub, so I imported the base images into Azure Container Registry with `az acr import`. A teammate's PR pointed the Dockerfile at the registry.

## Result

The chatbot returned real owner data from `customers-service` through function calling, and all six required test queries passed on AKS.

## The rest of the project

Other members of Group 9 built the Terraform infrastructure, the Helm chart, the Azure Pipeline and the monitoring. All of it is in the group's repo.

The `azure-pipelines.yml` in this repo isn't the pipeline that ran. The real one is in the group's repo.
