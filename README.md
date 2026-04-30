# datadog-ops

Repositório de configuração do Datadog Agent para clusters Kubernetes, gerenciado via Helm chart (`datadog/datadog`) e ArgoCD ApplicationSet.

## Estrutura de Pastas

```
datadog-ops/
├── argocd/
│   └── applicationset-prd.yaml     # Template ApplicationSet para clusters PRD
├── datadog/
│   └── values/
│       └── base.yaml           # Fonte única da verdade — configuração completa de produção
└── documents/
    ├── argocd-multienv-setup.md        # Guia de setup multi-cluster com ArgoCD ApplicationSet
    ├── datadog-architecture.drawio     # Diagrama de arquitetura (draw.io)
    └── datadog-architecture.drawio.png # Diagrama de arquitetura (PNG)
```

## Valores

| Arquivo | Descrição |
|---------|-----------|
| `base.yaml` | Fonte única da verdade — configuração completa e padrão para todos os clusters |

## Visão Geral

As configurações cobrem os seguintes componentes do Datadog:

- **Datadog Agent** — coleta de métricas, logs, APM e DogStatsD
- **Cluster Agent** — cluster checks, métricas customizadas via Datadog Metrics Provider
- **Trace Agent** — filtragem de health checks e coleta de traces APM
- **Process Agent** — coleta de processos em execução
- **OTLP Receiver** — ingestão de dados OpenTelemetry via gRPC e HTTP
- **Vector** — encaminhamento de logs

## Topologia GitOps — Multi-Cluster

Cada cluster tem seu próprio **ApplicationSet** no ArgoCD local, todos apontando para o `base.yaml` neste repositório:

```
datadog-ops (GitHub)
└── datadog/values/base.yaml   ← fonte única da verdade (Eng. Obs)
        ↑
        | todos os clusters referenciam o mesmo repo/branch
        |
├── ApplicationSet Cluster A  →  Datadog no Cluster A  →  Datadog SaaS
├── ApplicationSet Cluster B  →  Datadog no Cluster B  →  Datadog SaaS
└── ApplicationSet Cluster C  →  Datadog no Cluster C  →  Datadog SaaS
```

**Responsabilidades:**
- **Eng. Obs** — mantém `base.yaml` e o template ApplicationSet neste repositório
- **Time SRE** — copia o template para o ArgoCD de cada cluster; adiciona overrides específicos no `generators.list.elements`; não altera o `base.yaml`

## Uso

Copie o template ApplicationSet, aplique no ArgoCD do cluster e ajuste `clusterName`, `server` e `journey`:

```bash
kubectl apply -f argocd/applicationset-prd.yaml
```

Para adicionar overrides específicos de um cluster, declare-os no `generators.list.elements`:

```yaml
elements:
  - clusterName: cluster-b
    server: https://cluster-b-api:6443
    journey: minha-journey
    clusterAgentReplicas: "1"   # override do padrão definido em base.yaml
```

> As chaves `apiKey` e `appKey` são lidas de um Secret Kubernetes chamado `datadog` no namespace `datadog`. Crie o Secret antes de aplicar o ApplicationSet.

## Versões

- **Datadog Agent:** `7.73.0`
- **Cluster Agent:** `7.73.0`
