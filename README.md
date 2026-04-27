# datadog-ops

Repositório de configuração do Datadog Agent para clusters Kubernetes, gerenciado via Helm chart (`datadog/datadog`).

## Estrutura

| Arquivo | Descrição |
|---------|-----------|
| `base.yaml` | Valores base compartilhados entre todos os ambientes |
| `nprd.yaml` | Overrides para ambientes não-produção |
| `prod.yaml` | Overrides para ambientes de produção |

## Visão Geral

As configurações cobrem os seguintes componentes do Datadog:

- **Datadog Agent** — coleta de métricas, logs, APM e DogStatsD
- **Cluster Agent** — cluster checks, métricas customizadas via Datadog Metrics Provider
- **Trace Agent** — filtragem de health checks e coleta de traces APM
- **Process Agent** — coleta de processos em execução
- **OTLP Receiver** — ingestão de dados OpenTelemetry via gRPC e HTTP
- **Vector** — encaminhamento de logs (ativo em prod, desativado em nprd)

## Diferenças por Ambiente

| Feature | base | nprd | prod |
|---------|------|------|------|
| Log collection | `false` | `false` | `true` |
| APM | `false` | `false` | `true` |
| Cluster Checks | `true` | `false` | `true` |
| Vector logs | `true` | `false` | `true` |
| DatadogMonitors CRD | `true` | `false` | `true` |
| Log level | `INFO` | `WARN` | `WARN` |
| Container exclusions | mínimo | extenso | extenso |

## Uso

Os arquivos são aplicados via Helm com sobreposição de valores:

```bash
# Non-production
helm upgrade --install datadog datadog/datadog \
  -f base.yaml \
  -f nprd.yaml \
  --set datadog.apiKey=<API_KEY> \
  --set datadog.appKey=<APP_KEY> \
  --set datadog.clusterName=<CLUSTER_NAME>

# Production
helm upgrade --install datadog datadog/datadog \
  -f base.yaml \
  -f prod.yaml \
  --set datadog.apiKey=<API_KEY> \
  --set datadog.appKey=<APP_KEY> \
  --set datadog.clusterName=<CLUSTER_NAME>
```

> As chaves `apiKey` e `appKey` são injetadas via template (`{{apikey}}`, `{{appkey}}`) e devem ser fornecidas pelo pipeline de deploy ou gerenciador de segredos.

## Versões

- **Datadog Agent:** `7.73.0`
- **Cluster Agent:** `7.73.0`
