# AT – CI/CD e DevOps — Respostas e Evidências

**Imagem Docker Hub:** https://hub.docker.com/r/infnetfernando/ufotracker

## Missão 4 – Implantando a aplicação no cluster

### O que foi feito

No mesmo arquivo `ufodb-manifests.yaml`, adicionados:

- **ConfigMap** `app-config` com `DB_NAME: ufology`
- **Secret** `db-secret` com `DB_PASSWORD: devops2025!`
- **Deployment** `ufotracker` — 2 réplicas, imagem `infnetfernando/ufotracker:1.0`
- **Service** `ufotracker-service` — ClusterIP porta 8080

O `application.yaml` da aplicação foi configurado com variáveis de ambiente:

```yaml
host: ${REDIS_HOST:localhost}
url: jdbc:postgresql://${DB_HOST:localhost}:5432/${DB_NAME:ufology}
username: ${POSTGRES_USERNAME:postgres}
password: ${DB_PASSWORD:devops2025!}
```

### Manifesto: ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: ufology
data:
  DB_NAME: ufology
```

### Manifesto: Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: ufology
type: Opaque
stringData:
  DB_PASSWORD: "devops2025!"
```

### Manifesto: Deployment da aplicação

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ufotracker
  namespace: ufology
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ufotracker
  template:
    metadata:
      labels:
        app: ufotracker
    spec:
      containers:
      - name: ufotracker
        image: infnetfernando/ufotracker:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_NAME
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_NAME
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
        - name: DB_HOST
          value: ufodb-service
        - name: REDIS_HOST
          value: ufocache-service
        - name: POSTGRES_USERNAME
          value: postgres
```

### Resultado esperado no cluster

```
kubectl get all -n ufology

NAME                               READY   STATUS    RESTARTS
pod/ufodb-xxxx                     1/1     Running   0
pod/ufocache-xxxx                  1/1     Running   0
pod/ufotracker-xxxx-a              1/1     Running   0
pod/ufotracker-xxxx-b              1/1     Running   0

NAME                        TYPE        CLUSTER-IP   PORT(S)
service/ufodb-service       ClusterIP   ...          5432/TCP
service/ufocache-service    ClusterIP   ...          6379/TCP
service/ufotracker-service  ClusterIP   ...          8080/TCP
```

---

## Parte 2 – Workflows Básicos no GitHub Actions

Todos os workflows estão em `.github/workflows/` na raiz do repositório.  
🔗 https://github.com/lntra/AT_CICD_/actions

---

### `hello.yml` — Hello CI/CD

**Disparo:** qualquer `push`

```yaml
name: Hello Workflow

on:
  push:

jobs:
  hello:
    runs-on: ubuntu-latest
    steps:
      - name: Exibir mensagem
        run: echo "Hello CI/CD"
```

**Log esperado:**
```
Hello CI/CD
```

---

### `tests.yml` — Rodando testes

**Disparo:** `pull_request`

```yaml
name: Tests Workflow

on:
  pull_request:

jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout do código
        uses: actions/checkout@v4

      - name: Rodando testes
        run: echo "Rodando testes"
```

**Log esperado:**
```
Rodando testes
```

> **Para disparar:** crie um Pull Request em https://github.com/lntra/AT_CICD_

---

### `gradle-ci.yml` — Build CI

**Disparo:** push na branch `main`

```yaml
name: Build CI

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout do código
        uses: actions/checkout@v4

      - name: Configurar Java
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'

      - name: Build (Gradle ou Maven)
        run: |
          cd ufoTracker
          if [ -f ./gradlew ]; then
            chmod +x ./gradlew
            ./gradlew build -x test
          else
            chmod +x ./mvnw
            ./mvnw -B clean package -DskipTests
          fi
```

---

## Parte 3 – Runners, Variáveis e Segurança

### `env-demo.yml` — Variável de ambiente

**Disparo:** qualquer `push`

```yaml
name: env-demo

on:
  push:

env:
  DEPLOY_ENV: staging

jobs:
  show-env:
    runs-on: ubuntu-latest
    steps:
      - name: Exibir variável de ambiente
        run: |
          echo "Ambiente de deploy configurado como: $DEPLOY_ENV"

      - name: Confirmar valor via env block
        env:
          DEPLOY_ENV: ${{ env.DEPLOY_ENV }}
        run: |
          echo "DEPLOY_ENV=$DEPLOY_ENV"
```

**Log esperado:**
```
Ambiente de deploy configurado como: staging
DEPLOY_ENV=staging
```

---

### `secret-demo.yml` — Secret seguro

**Disparo:** qualquer `push`

**Pré-requisito:** configurar o secret `API_KEY` em:  
https://github.com/lntra/AT_CICD_/settings/secrets/actions → _New repository secret_

```yaml
name: secret-demo

on:
  push:

jobs:
  show-secret:
    runs-on: ubuntu-latest
    steps:
      - name: Verificar presença do secret API_KEY
        env:
          API_KEY: ${{ secrets.API_KEY }}
        run: |
          if [ -n "$API_KEY" ]; then
            echo "API_KEY configurado"
          else
            echo "API_KEY não encontrado ou vazio"
          fi
```

**Log esperado (com secret configurado):**
```
API_KEY configurado
```

> O valor real do secret **nunca aparece no log**. O GitHub automaticamente mascara qualquer valor de secret como `***`. A lógica condicional (`if [ -n "$API_KEY" ]`) apenas confirma a presença sem expor o conteúdo.

---

### Diferença entre Runners hospedados pelo GitHub e Auto-hospedados

#### Runners hospedados pelo GitHub (_GitHub-hosted runners_)

São máquinas virtuais provisionadas e gerenciadas pelo próprio GitHub. A cada execução de workflow, uma VM limpa é criada, usada e descartada.

| Característica | Detalhe |
|---|---|
| **SO disponíveis** | Ubuntu, Windows, macOS |
| **Manutenção** | GitHub cuida de atualizações, patches e ferramentas pré-instaladas |
| **Custo** | Gratuito até certo limite de minutos por mês (dependendo do plano) |
| **Isolamento** | VM limpa a cada job — sem contaminação entre execuções |
| **Configuração** | Zero configuração necessária |
| **Acesso à rede interna** | Não — apenas acesso à internet pública |

**Vantagens:**
- Pronto para uso imediato, sem infraestrutura própria
- Ambiente limpo e reproduzível garantido
- Suporta Ubuntu, Windows e macOS nativamente
- Ideal para projetos open-source e equipes pequenas

**Desvantagens:**
- Limite de minutos gratuitos (contas pessoais: 2.000 min/mês)
- Sem acesso a redes privadas, bancos internos ou VPNs corporativas
- Hardware fixo (não customizável)
- Pode ser mais lento por conta de fila de execução

---

#### Runners auto-hospedados (_Self-hosted runners_)

São máquinas próprias (físicas ou VMs) registradas no repositório do GitHub. O agente do GitHub Actions é instalado na máquina e ela passa a receber jobs.

| Característica | Detalhe |
|---|---|
| **Hardware** | Customizável — pode usar GPU, SSD NVMe, mais RAM, etc. |
| **Manutenção** | Responsabilidade do usuário/empresa |
| **Custo** | Sem cobrança de minutos — apenas custo da infraestrutura própria |
| **Acesso à rede** | Total — acessa bancos internos, VPNs, serviços corporativos |
| **Ambiente** | Persistente entre execuções (pode haver contaminação de estado) |

**Vantagens:**
- Sem limite de minutos de execução
- Acesso completo à infraestrutura interna (banco de dados, registros privados, etc.)
- Hardware dedicado — pode ser significativamente mais rápido para builds pesados
- Controle total do ambiente de execução
- Útil para compilar projetos que precisam de dependências específicas ou licenças de software

**Desvantagens:**
- Manutenção e atualização são responsabilidade da equipe
- Se o ambiente não for limpo entre jobs, pode haver efeitos colaterais
- Risco de segurança maior: código de PRs externos pode executar na sua máquina
- Requer setup inicial e monitoramento contínuo

---

## Checklist Final

| Parte 3 – Secret `API_KEY` no repositório | ✅ | Configurado e validado no check `secret-demo / show-secret (push)` |

