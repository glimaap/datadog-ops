# ArgoCD — Multi-Cluster com ApplicationSet e Values Compartilhados

Guia para configurar o Datadog Agent em múltiplos clusters Kubernetes via ArgoCD ApplicationSet, com um único repositório de values compartilhado entre todos os clusters.

---

## Visão Geral da Arquitetura

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

---

## Pré-requisitos

- ArgoCD >= 2.6 instalado em cada cluster (suporte a `sources` múltiplos e `goTemplate`)
- Repositório Git com os values files acessível pelo ArgoCD de cada cluster
- `kubectl` configurado para o cluster alvo
- Secret com a API key do Datadog criado no namespace `datadog`

---

## Passo 1 — Estrutura do Repositório

```
datadog-ops/
├── argocd/
│   └── applicationset-prd.yaml   # template ApplicationSet
└── datadog/
    └── values/
        └── base.yaml             # configuração completa — padrão para todos os clusters
```

---

## Passo 2 — Criar o Secret da API Key

Crie o Secret no cluster **antes** de aplicar o ApplicationSet:

```bash
kubectl create namespace datadog

kubectl create secret generic datadog \
  --from-literal=api-key=<SUA_API_KEY> \
  -n datadog
```

> **Importante:** Este Secret não é gerenciado pelo ArgoCD. Se for deletado, recrie manualmente ou use ExternalSecrets/SealedSecrets.

---

## Passo 3 — Entendendo o ApplicationSet

O `ApplicationSet` substitui os `Application` individuais. Com ele, um único manifest no ArgoCD gera automaticamente uma `Application` por entrada no `generators.list.elements`.

**Por que `goTemplate: true`?**

Habilita sintaxe Go template nas variáveis, permitindo valores padrão com `{{ default "valor" .parametro }}`. Assim, overrides por cluster são opcionais — se não declarados no elemento, o valor do `base.yaml` prevalece.

**Por que `sources` (plural)?**

Necessário para combinar o chart Helm externo (`helm.datadoghq.com`) com os values files do repositório Git. O `ref: values` cria o alias `$values` que o primeiro source usa para localizar os arquivos.

---

## Passo 4 — Template ApplicationSet

Arquivo: `argocd/applicationset-prd.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: datadog-prd
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=zero"]
  generators:
    - list:
        elements:
          # Um elemento por cluster
          # Campos obrigatórios: clusterName, server, journey
          - clusterName: cluster-a
            server: https://cluster-a-api:6443
            journey: minha-journey
          # Exemplo com override por cluster:
          # - clusterName: cluster-b
          #   server: https://cluster-b-api:6443
          #   journey: outra-journey
          #   clusterAgentReplicas: "1"
  template:
    metadata:
      name: 'datadog-{{.clusterName}}'
    spec:
      project: default
      sources:
        - repoURL: https://helm.datadoghq.com
          chart: datadog
          targetRevision: "*"
          helm:
            valueFiles:
              - $values/datadog/values/base.yaml
            valuesObject:
              datadog:
                apiKeyExistingSecret: datadog
                apiKeyExistingSecretKey: api-key
                appKey: ""
                clusterName: '{{.clusterName}}'
                tags:
                  - env:prd
                  - 'journey:{{.journey}}'
              # Overrides opcionais por cluster:
              # clusterAgent:
              #   replicas: '{{ default "3" .clusterAgentReplicas }}'
        - repoURL: https://github.com/<ORG>/<REPO>.git
          targetRevision: main
          ref: values
      destination:
        server: '{{.server}}'
        namespace: datadog
      ignoreDifferences:
        - kind: ConfigMap
          name: 'datadog-{{.clusterName}}-confd'
          namespace: datadog
          jsonPointers:
            - /data
        - kind: ConfigMap
          name: 'datadog-{{.clusterName}}-cluster-agent-confd'
          namespace: datadog
          jsonPointers:
            - /data
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - RespectIgnoreDifferences=true
```

---

## Passo 5 — Adicionar um Novo Cluster

Edite o `generators.list.elements` do ApplicationSet no ArgoCD do cluster:

```yaml
elements:
  - clusterName: cluster-a
    server: https://cluster-a-api:6443
    journey: payments
  # Novo cluster:
  - clusterName: cluster-d
    server: https://cluster-d-api:6443
    journey: catalog
```

O ArgoCD detecta a mudança e cria automaticamente a Application `datadog-cluster-d`.

---

## Passo 6 — Overrides por Cluster

Para customizar o Datadog de um cluster específico sem alterar o `base.yaml`, adicione o parâmetro no elemento e descomente o override no `valuesObject` do template:

**No elemento do gerador:**
```yaml
- clusterName: cluster-b
  server: https://cluster-b-api:6443
  journey: checkout
  clusterAgentReplicas: "1"
```

**No template (valuesObject):**
```yaml
valuesObject:
  clusterAgent:
    replicas: '{{ default "3" .clusterAgentReplicas }}'
```

O `default "3"` garante que clusters sem esse parâmetro continuem usando o valor do `base.yaml`.

---

## Passo 7 — Aplicar o ApplicationSet

```bash
kubectl apply -f argocd/applicationset-prd.yaml
```

Acompanhe o sync:

```bash
kubectl get applicationset -n argocd
kubectl get application -n argocd
kubectl get pods -n datadog -w
```

---

## Fluxo GitOps — Como Fazer uma Mudança

**Mudança que afeta todos os clusters** (ex: atualizar versão do agent):
1. Edite `datadog/values/base.yaml`
2. Abra PR para `main`
3. Após merge, todos os ApplicationSets sincronizam automaticamente

**Mudança específica de um cluster** (ex: override de réplicas):
1. Adicione o parâmetro no `generators.list.elements` do ApplicationSet local
2. Descomente o override no `valuesObject` do template
3. `kubectl apply` no ArgoCD do cluster (não requer PR neste repositório)

```
base.yaml atualizado → PR → Merge em main → ArgoCD sync em todos os clusters
```

---

## Problemas Comuns

### Placeholders `{{variavel}}` nos values files

O Helm interpreta `{{` como sintaxe de template Go. Use `valuesObject` no ApplicationSet para passar valores dinâmicos — nunca deixe placeholders no `base.yaml`.

### `InvalidImageName` nos pods

Separe registry, name e tag nos campos corretos:

```yaml
agents:
  image:
    registry: gcr.io/datadoghq
    name: agent
    tag: "7.73.0"
```

### Secret da API key deletado

```bash
kubectl create secret generic datadog \
  --from-literal=api-key=<SUA_API_KEY> \
  -n datadog
```

### Mudança no base.yaml afeta clusters indesejados

Para mudanças de alto risco, desative o sync automático temporariamente:

```yaml
syncPolicy:
  automated: null   # sync manual até validar a mudança
```
