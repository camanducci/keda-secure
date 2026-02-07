# KEDA Secure - Imagens sem Vulnerabilidades

Este repositório contém o KEDA configurado com imagens Docker seguras, livres de vulnerabilidades HIGH e CRITICAL.

## Problema

As imagens oficiais do KEDA 2.16.1 possuem vulnerabilidades:

| Imagem Oficial | Vulnerabilidades HIGH |
|----------------|----------------------|
| `ghcr.io/kedacore/keda:2.16.1` | 12 |
| `ghcr.io/kedacore/keda-metrics-apiserver:2.16.1` | 12 |
| `ghcr.io/kedacore/keda-admission-webhooks:2.16.1` | 9 |

## Solução

Imagens reconstruídas com Go 1.25.6 e dependências atualizadas:

| Imagem Segura | Vulnerabilidades |
|---------------|------------------|
| `camanducci/keda:local-secure` | **0** ✅ |
| `camanducci/keda-metrics-apiserver:local-secure` | **0** ✅ |
| `camanducci/keda-admission-webhooks:local-secure` | **0** ✅ |

---

## CI/CD Pipeline

Este repositório utiliza GitHub Actions para automatizar o processo de build, scan e deploy das imagens.

### Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         Pipeline                                 │
├─────────────────────────────────────────────────────────────────┤
│  1. scan-dependencies     → govulncheck nas dependências Go     │
│           ↓                                                      │
│  2. build-and-scan        → Build + Trivy + Grype (3 imagens)   │
│           ↓                                                      │
│  3. push-images           → Push para Docker Hub (só em main)   │
│           ↓                                                      │
│  4. security-report       → Gera resumo no GitHub Summary       │
└─────────────────────────────────────────────────────────────────┘
```

### Etapas do Pipeline

| Etapa | Descrição | Ferramentas |
|-------|-----------|-------------|
| **scan-dependencies** | Analisa vulnerabilidades nas dependências Go | govulncheck |
| **build-and-scan** | Constrói as 3 imagens e escaneia cada uma | Docker, Trivy, Grype |
| **push-images** | Publica imagens no Docker Hub (apenas branch main) | Docker Hub |
| **security-report** | Gera relatório de segurança no GitHub Summary | GitHub Actions |

### Configurar Secrets

Para o pipeline funcionar, configure os secrets no GitHub:

1. Acesse: `Settings → Secrets and variables → Actions`
2. Adicione os seguintes secrets:

| Secret | Descrição |
|--------|-----------|
| `DOCKERHUB_USERNAME` | Seu usuário do Docker Hub |
| `DOCKERHUB_TOKEN` | Token de acesso do Docker Hub |

### Criar Token no Docker Hub

1. Acesse: https://hub.docker.com/settings/security
2. Clique em **New Access Token**
3. Nome: `github-actions`
4. Permissões: **Read & Write**
5. Copie o token e adicione no GitHub Secrets

### Executar Pipeline

O pipeline é executado automaticamente em:
- ✅ Push na branch `main`
- ✅ Pull Requests para `main`
- ✅ Manualmente via **Actions → Run workflow**

### Política de Segurança

O pipeline **falha automaticamente** se encontrar:
- Vulnerabilidades **HIGH** ou **CRITICAL** nas imagens
- Vulnerabilidades nas dependências Go

---

## Passo a Passo (Manual)

### 1. Escanear Vulnerabilidades

```bash
# Instalar Trivy (scanner de vulnerabilidades)
# https://trivy.dev/

# Escanear imagem oficial
trivy image --severity HIGH,CRITICAL ghcr.io/kedacore/keda:2.16.1

# Escanear dependências Go do projeto
docker run --rm -v $(pwd):/app -w /app golang:1.25 sh -c \
  "go install golang.org/x/vuln/cmd/govulncheck@latest && govulncheck ./..."
```

### 2. Construir Imagens Seguras

```bash
# Build da imagem principal (keda operator)
docker build -t camanducci/keda:local-secure -f Dockerfile .

# Build do metrics-apiserver
docker build -t camanducci/keda-metrics-apiserver:local-secure -f Dockerfile.adapter .

# Build do admission-webhooks
docker build -t camanducci/keda-admission-webhooks:local-secure -f Dockerfile.webhooks .
```

### 3. Verificar Segurança das Novas Imagens

```bash
# Escanear cada imagem
trivy image --severity HIGH,CRITICAL camanducci/keda:local-secure
trivy image --severity HIGH,CRITICAL camanducci/keda-metrics-apiserver:local-secure
trivy image --severity HIGH,CRITICAL camanducci/keda-admission-webhooks:local-secure

# Ou usar Grype (alternativa)
grype camanducci/keda:local-secure
```

### 4. Enviar para Docker Hub

```bash
docker login

docker push camanducci/keda:local-secure
docker push camanducci/keda-metrics-apiserver:local-secure
docker push camanducci/keda-admission-webhooks:local-secure
```

### 5. Instalar no Kubernetes

```bash
# Criar cluster de teste (kind)
kind create cluster --name keda-test

# Aplicar configurações do KEDA
kubectl apply -k config/default

# Verificar pods
kubectl get pods -n keda
```

---

## Demo: KEDA + RabbitMQ

### Arquivos de Demonstração

```
demo/
├── rabbitmq-demo.yaml           # RabbitMQ + ScaledObject
├── rabbitmq-publisher.yaml      # Publisher de mensagens
├── rabbitmq-consumer-fixed.yaml # Consumer Python
├── monitoring.yaml              # ServiceMonitors Prometheus
└── grafana-dashboard.yaml       # Dashboard Grafana
```

### Instalar a Demo

```bash
# Criar namespace
kubectl create namespace rabbitmq-demo

# Aplicar RabbitMQ e ScaledObject
kubectl apply -f demo/rabbitmq-demo.yaml

# Aguardar RabbitMQ ficar pronto
kubectl wait --for=condition=ready pod -l app=rabbitmq -n rabbitmq-demo --timeout=120s

# Verificar estado inicial (0 consumers)
kubectl get deployment rabbitmq-consumer -n rabbitmq-demo
```

### Testar Escalonamento Automático

```bash
# Publicar 50 mensagens na fila
kubectl apply -f demo/rabbitmq-publisher.yaml

# Observar KEDA escalando os consumers
kubectl get pods -n rabbitmq-demo -l app=rabbitmq-consumer -w

# Verificar HPA
kubectl get hpa -n rabbitmq-demo
```

### Resultado Esperado

1. **Antes**: 0 réplicas (scale-to-zero)
2. **Após publicar mensagens**: 8-10 réplicas (scale-up)
3. **Após consumir mensagens**: 0 réplicas (scale-down)

---

## Monitoramento com Prometheus + Grafana

### Instalar Stack de Monitoramento

```bash
# Adicionar repo Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Instalar Prometheus + Grafana
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin123

# Aplicar ServiceMonitors
kubectl apply -f demo/monitoring.yaml

# Habilitar métricas no RabbitMQ
kubectl exec -n rabbitmq-demo deployment/rabbitmq -- rabbitmq-plugins enable rabbitmq_prometheus

# Aplicar dashboard Grafana
kubectl apply -f demo/grafana-dashboard.yaml
```

### Acessar Grafana

```bash
# Port-forward
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Acessar
# URL: http://localhost:3000
# Usuário: admin
# Senha: admin123

# Dashboard: Dashboards > Browse > KEDA - RabbitMQ Scaling
```

---

## Arquivos Modificados

| Arquivo | Alteração |
|---------|-----------|
| `config/manager/manager.yaml` | Imagem: `camanducci/keda:local-secure` |
| `config/metrics-server/deployment.yaml` | Imagem: `camanducci/keda-metrics-apiserver:local-secure` |
| `config/webhooks/webhooks.yaml` | Imagem: `camanducci/keda-admission-webhooks:local-secure` |

---

## Ferramentas Utilizadas

| Ferramenta | Uso |
|------------|-----|
| [Trivy](https://trivy.dev/) | Scanner de vulnerabilidades |
| [Grype](https://github.com/anchore/grype) | Scanner alternativo |
| [govulncheck](https://pkg.go.dev/golang.org/x/vuln/cmd/govulncheck) | Scanner de vulnerabilidades Go |
| [kind](https://kind.sigs.k8s.io/) | Cluster Kubernetes local |
| [KEDA](https://keda.sh/) | Kubernetes Event-driven Autoscaling |

---

## Referências

- [KEDA Documentation](https://keda.sh/docs/)
- [KEDA RabbitMQ Scaler](https://keda.sh/docs/latest/scalers/rabbitmq-queue/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Docker Scout](https://docs.docker.com/scout/)

---

## Autor

**camanducci**
