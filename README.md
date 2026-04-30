# datadog-ops

Repositório de configuração do Datadog Agent para clusters Kubernetes, gerenciado via Helm chart (`datadog/datadog`).

## Estrutura de Pastas

```
datadog-ops/
├── argocd/
│   ├── datadog-nprd.yaml       # ArgoCD Application para ambientes não-produção
│   └── datadog-prd.yaml        # ArgoCD Application para produção
├── datadog/
│   └── values/
│       ├── base.yaml           # Fonte única da verdade — valores completos de produção
│       └── nprd.yaml           # Overrides de nprd (apenas 6 diferenças em relação ao base)
└── documents/
    ├── argocd-multienv-setup.md        # Guia de setup multi-ambiente com ArgoCD
    ├── datadog-architecture.drawio     # Diagrama de arquitetura (draw.io)
    └── datadog-architecture.drawio.png # Diagrama de arquitetura (PNG)
```

## Valores

| Arquivo | Descrição |
|---------|-----------|
| `base.yaml` | Configuração completa e fonte única da verdade — reflete os valores de produção |
| `nprd.yaml` | Overrides para ambientes não-produção (apenas os 6 valores que diferem de prod) |

> `prod.yaml` não existe. Ambientes de produção usam somente `base.yaml`.

## Visão Geral

As configurações cobrem os seguintes componentes do Datadog:

- **Datadog Agent** — coleta de métricas, logs, APM e DogStatsD
- **Cluster Agent** — cluster checks, métricas customizadas via Datadog Metrics Provider
- **Trace Agent** — filtragem de health checks e coleta de traces APM
- **Process Agent** — coleta de processos em execução
- **OTLP Receiver** — ingestão de dados OpenTelemetry via gRPC e HTTP
- **Vector** — encaminhamento de logs (ativo em prod, desativado em nprd)

## Diferenças por Ambiente

`nprd.yaml` sobrescreve apenas os valores abaixo. Tudo mais herda de `base.yaml`:

| Propriedade | base (prod) | nprd |
|-------------|-------------|------|
| `datadog.clusterChecks.enabled` | `true` | `false` |
| `clusterAgent.replicas` | `3` | `1` |
| `clusterAgent.createPodDisruptionBudget` | `true` | `false` |
| `agents.customAgentConfig.vector.logs.enabled` | `true` | `false` |
| `datadog-crds.crds.datadogMonitors` | `true` | `false` |
| `datadog-crds.crds.datadogMetrics` | `true` | `false` |

## Uso

Os arquivos são aplicados via Helm com sobreposição de valores:

```bash
# Production
helm upgrade --install datadog datadog/datadog \
  -f base.yaml \
  --set datadog.apiKey=<API_KEY> \
  --set datadog.appKey=<APP_KEY> \
  --set datadog.clusterName=<CLUSTER_NAME>

# Non-production
helm upgrade --install datadog datadog/datadog \
  -f base.yaml \
  -f nprd.yaml \
  --set datadog.apiKey=<API_KEY> \
  --set datadog.appKey=<APP_KEY> \
  --set datadog.clusterName=<CLUSTER_NAME>
```

> As chaves `apiKey` e `appKey` são injetadas via template (`{{apikey}}`, `{{appkey}}`) e devem ser fornecidas pelo pipeline de deploy ou gerenciador de segredos.

## Versões

- **Datadog Agent:** `7.73.0`
- **Cluster Agent:** `7.73.0`
