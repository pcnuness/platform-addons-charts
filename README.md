# Platform Addons Charts

Repositório centralizado de Helm Charts para addons de plataforma Kubernetes.

## 📁 Estrutura

```
platform-addons-charts/
├── addons/
│   ├── metrics-server/
│   │   ├── Chart.yaml          # Definição do chart + versão
│   │   └── values.yaml         # Valores padrão base
│   ├── grafana/
│   │   ├── Chart.yaml
│   │   └── values.yaml
│   └── crossplane/
│       ├── Chart.yaml
│       └── values.yaml
└── README.md
```

## 🎯 Princípios

### 1. **Centralização**
- Um único Chart por addon
- Valores padrão aplicáveis a todos os ambientes
- Versionamento controlado via `Chart.yaml`

### 2. **Customização por Cluster**
- Valores específicos são definidos no repositório `gitops-cluster-management`
- Cada cluster pode sobrescrever qualquer valor padrão
- Separação clara entre padrões e customizações

### 3. **Versionamento**
Cada addon possui duas versões no `Chart.yaml`:
- **`version`**: Versão do wrapper chart (nosso controle)
- **`appVersion`**: Versão da aplicação upstream
- **`dependencies[].version`**: Versão exata do chart upstream

## 📦 Addons Disponíveis

### Metrics Server
- **Chart Version**: 0.1.0
- **App Version**: 3.12.2
- **Upstream**: https://kubernetes-sigs.github.io/metrics-server/
- **Descrição**: Coleta métricas de recursos do cluster

### Grafana
- **Chart Version**: 0.1.0
- **App Version**: 10.0.0
- **Upstream**: https://grafana.github.io/helm-charts
- **Descrição**: Plataforma de observabilidade e dashboards

### Crossplane
- **Chart Version**: 0.1.0
- **App Version**: 1.15.3
- **Upstream**: https://charts.crossplane.io/stable
- **Descrição**: Infrastructure as Code para provisionamento de recursos cloud

## 🔄 Estratégias de Rollout

### 1. **Versionamento Semântico**

Utilize tags Git para versionar releases:

```bash
# Criar uma nova versão
git tag -a v0.1.0 -m "Initial release: metrics-server 3.12.2, grafana 10.0.0"
git push origin v0.1.0

# Atualizar versão de um addon
git tag -a v0.2.0 -m "Update metrics-server to 3.13.0"
git push origin v0.2.0
```

### 2. **Rollout Gradual por Ambiente**

No `gitops-cluster-management`, controle qual versão cada ambiente usa:

```yaml
# ApplicationSet
spec:
  template:
    spec:
      sources:
        - repoURL: 'https://github.com/pcnuness/platform-addons-charts.git'
          # Develop usa 'main' (bleeding edge)
          targetRevision: 'main'
          
        # UAT usa versão específica (stable)
        # targetRevision: 'v0.1.0'
        
        # Prod usa versão testada (production-ready)
        # targetRevision: 'v0.1.0'
```

### 3. **Estratégias de Rollout por Addon**

#### A. Blue-Green Deployment
```yaml
# gitops-cluster-management/addons/metrics-server/values.yaml
metrics-server:
  fullnameOverride: metrics-server-v2  # Nova versão
  replicas: 2
  # Manter versão antiga rodando até validar
```

#### B. Canary Deployment
```yaml
# ApplicationSet com weight
spec:
  template:
    spec:
      syncPolicy:
        automated:
          prune: false  # Não remover versão antiga automaticamente
        syncOptions:
          - CreateNamespace=true
          - RespectIgnoreDifferences=true
```

#### C. Rolling Update (Padrão)
```yaml
metrics-server:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

### 4. **Controle de Versão Granular**

#### Por Cluster Label
```yaml
# ApplicationSet
generators:
  - clusters:
      selector:
        matchLabels:
          version-metrics-server: "3.12.2"  # Cluster específico
```

#### Por Annotation
```yaml
# ApplicationSet
template:
  metadata:
    annotations:
      addon.version/metrics-server: "{{metadata.annotations.metrics-server-version}}"
```

### 5. **Rollback Rápido**

```bash
# Via Git
git revert <commit-hash>
git push

# Via ArgoCD CLI
argocd app rollback addon-oss-metrics-server-data-plataform-dev-eks <history-id>

# Via kubectl
kubectl rollout undo deployment/metrics-server -n kube-system
```

## 🚀 Workflow de Atualização

### Cenário: Atualizar Metrics Server de 3.12.2 para 3.13.0

#### 1. **Atualizar Chart Central**
```bash
cd platform-addons-charts/addons/metrics-server

# Editar Chart.yaml
vim Chart.yaml
# version: 0.2.0
# appVersion: "3.13.0"
# dependencies[].version: 3.13.0

git add .
git commit -m "feat(metrics-server): upgrade to 3.13.0"
git tag -a v0.2.0 -m "Metrics Server 3.13.0"
git push origin main --tags
```

#### 2. **Testar em Develop**
```yaml
# gitops-cluster-management ApplicationSet já usa 'main'
# ArgoCD detecta mudança automaticamente
# Validar por 24-48h
```

#### 3. **Promover para UAT**
```yaml
# Atualizar ApplicationSet para UAT
targetRevision: 'v0.2.0'  # Versão testada
```

#### 4. **Promover para Prod**
```yaml
# Após validação em UAT (1 semana)
targetRevision: 'v0.2.0'
```

## 🔐 Boas Práticas

1. **Sempre use tags para produção**
   - `main` = desenvolvimento
   - `v0.x.x` = produção

2. **Documente mudanças**
   - Mantenha CHANGELOG.md
   - Use conventional commits

3. **Teste antes de promover**
   - Develop → UAT → Prod
   - Mínimo 48h em cada ambiente

4. **Monitore após deploy**
   - Métricas de recursos
   - Logs de erros
   - Health checks

5. **Tenha plano de rollback**
   - Documente versão anterior
   - Teste rollback em develop

## 📊 Monitoramento de Versões

```bash
# Ver versões deployadas
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}'

# Via ArgoCD
argocd app list -o wide | grep addon-oss
```

## 🔗 Referências

- [ArgoCD Multi-Source Apps](https://argo-cd.readthedocs.io/en/stable/user-guide/multiple_sources/)
- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
