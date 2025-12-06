# Sample Web App - Kubernetes Manifests

Helm chart para deploy da aplicação Echo Web App usando ArgoCD e Argo Rollouts com Kong Ingress.

## 📁 Estrutura

```
.
├── argocd/
│   └── application.yaml      # ArgoCD Application manifest
└── charts/
    └── sample-web-app/
        ├── Chart.yaml         # Metadata do chart
        ├── values.yaml        # Valores padrão
        ├── values-dev.yaml    # Valores para dev
        ├── values-prod.yaml   # Valores para produção
        └── templates/
            ├── _helpers.tpl   # Template helpers
            ├── configmap.yaml # ConfigMap
            ├── hpa.yaml       # HorizontalPodAutoscaler
            ├── ingress.yaml   # Ingress (Kong)
            ├── rollout.yaml   # Argo Rollout/Deployment
            ├── service.yaml   # Services (stable/canary)
            └── serviceaccount.yaml
```

## 🚀 Deploy com ArgoCD

### 1. Aplicar o Application no ArgoCD

```bash
kubectl apply -f argocd/application.yaml
```

### 2. Ou via ArgoCD CLI

```bash
argocd app create sample-web-app \
  --repo https://github.com/gabrielsilvao/challenge1-k8s-manifests.git \
  --path charts/sample-web-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace sample-web-app \
  --helm-set image.repository=<ECR_URL>/sample-web-app \
  --helm-set image.tag=latest
```

## 🔄 Argo Rollouts - Canary Deployment

A aplicação usa Argo Rollouts para canary deployments:

### Estratégia padrão (dev):
1. 50% do tráfego para canary
2. Pausa de 30 segundos
3. 100% (promoção completa)

### Estratégia produção:
1. 10% → pausa 1min
2. 30% → pausa 2min
3. 50% → pausa 2min
4. 70% → pausa 2min
5. 90% → pausa 5min
6. 100% (promoção completa)

### Comandos úteis

```bash
# Ver status do rollout
kubectl argo rollouts get rollout sample-web-app -n sample-web-app

# Promover canary manualmente
kubectl argo rollouts promote sample-web-app -n sample-web-app

# Abortar rollout
kubectl argo rollouts abort sample-web-app -n sample-web-app

# Dashboard do Argo Rollouts
kubectl argo rollouts dashboard
```

## 🌐 Kong Ingress

O chart configura um Ingress usando Kong como Ingress Controller:

- **IngressClass**: `kong`
- **Annotations disponíveis**:
  - `konghq.com/strip-path`: Remove o path antes de enviar ao backend
  - `konghq.com/protocols`: Protocolos aceitos (http, https)
  - `konghq.com/https-redirect-status-code`: Código de redirecionamento para HTTPS

## ⚙️ Configuração

### Atualizar URL do ECR

Edite `values-dev.yaml` e `values-prod.yaml`:

```yaml
image:
  repository: <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/sample-web-app
```

### Variáveis de ambiente

```yaml
env:
  - name: PORT
    value: "8080"
  - name: CUSTOM_VAR
    value: "value"
```

## 📊 Monitoramento

### Health Checks

- **Liveness**: `/health` - Verifica se a aplicação está viva
- **Readiness**: `/health` - Verifica se está pronta para receber tráfego

### Recursos

| Ambiente | CPU Request | CPU Limit | Memory Request | Memory Limit |
|----------|-------------|-----------|----------------|--------------|
| Dev      | 25m         | 100m      | 32Mi           | 128Mi        |
| Prod     | 100m        | 500m      | 128Mi          | 512Mi        |

## 🔒 Segurança

- Executa como usuário não-root (UID 1000)
- ReadOnly filesystem
- Sem privilege escalation
- Capabilities dropped
