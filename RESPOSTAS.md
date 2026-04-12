# AT – CI/CD e DevOps — Respostas e Evidências

**Imagem Docker Hub:** https://hub.docker.com/r/infnetfernando/ufotracker

## Missão 4 – Implantando a aplicação no cluster

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

**Log:**

![hello.yml log](Images/image1.png)

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

**Log:**

![tests.yml log](Images/image2.png)

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

**Log:**

![env-demo log](Images/image4.png)

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

**Log:**

![secret-demo log](Images/image3.png)

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


