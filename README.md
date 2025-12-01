# Controle de Estoque

## Descrição
Aplicação back-end em Spring Boot com banco de dados PostgreSQL, rodando em Docker. Esta aplicação permite a comunicação cliente-servidor através da criação de APIs, onde o cliente é a aplicação web e o servidor armazena o banco de dados com as informações do sistema.

## Requisitos
- Docker e Docker Compose instalados no computador.
- (Opcional) VS Code com a extensão Dev Containers para desenvolvimento em contêiner.

## Como Rodar a Aplicação
1. Copie o arquivo `.env` de exemplo (já presente) e ajuste as variáveis se necessário:
   - `APP_PORT=8080`
   - `POSTGRES_PORT=5432`
   - `POSTGRES_DB=controleestoque`
   - `POSTGRES_USER=root`
   - `POSTGRES_PASSWORD=root`
2. No terminal (CMD/PowerShell) na pasta do projeto, execute:
   ```bash
   docker compose up --build
   ```
3. A API estará disponível em `http://localhost:8080` e o banco em `localhost:5432`.

### Swagger / OpenAPI
- A documentação interativa está disponível em: `http://localhost:8080/swagger-ui.html`
- Documento OpenAPI (JSON): `http://localhost:8080/v3/api-docs`
- Os endpoints documentados usam o prefixo `/api/**`.

### Exemplos de Requisições (cURL)
1) Autenticação (obter token JWT):
```bash
curl -X POST "http://localhost:8080/api/auth/sign-in" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin"}'
```
Resposta esperada (exemplo):
```json
{
  "authenticated": true,
  "created": "2025-01-01T00:00:00Z",
  "expiration": "2025-01-01T01:00:00Z",
  "accessToken": "<JWT>",
  "refreshToken": "<JWT_REFRESH>"
}
```
2) Usando o token para acessar um endpoint protegido (ex.: listar pessoas):
```bash
ACCESS_TOKEN="<cole o token aqui>"
curl -X GET "http://localhost:8080/api/pessoa/" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"
```
3) Criar uma pessoa (exemplo):
```bash
ACCESS_TOKEN="<cole o token aqui>"
curl -X POST "http://localhost:8080/api/pessoa/create" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
        "nome": "Fulano de Tal",
        "cpf": "12345678901",
        "email": "fulano@example.com"
      }'
```

## Docker e Imagens
- Dockerfile multi-stage implementado para a aplicação (`docker-config/app/Dockerfile`):
  - Stage de build (Maven + JDK 19) gera o `.jar`.
  - Stage de runtime usa somente JRE 19 para reduzir o tamanho da imagem.
- A imagem do banco usa diretamente `postgres:17-alpine` via `docker-compose.yml` (não é necessário um Dockerfile próprio só para expor porta).

## Dev Container (opcional)
- Foi adicionado `.devcontainer/devcontainer.json` com Java 19 + Maven.
- Para usar no VS Code: abrir a pasta do projeto e selecionar "Reopen in Container".
- Portas encaminhadas: 8080 (API) e 5432 (Postgres).

## Observações
- A aplicação lê as configurações de banco via variáveis do ambiente definidas no `docker-compose.yml` (por exemplo `SPRING_DATASOURCE_URL`), apontando para o serviço `postgres-service`.
- O arquivo `docker-config/db/Dockerfile` tornou-se opcional e não é mais referenciado pelo Compose atual.

# Tecnologias Utilizadas
## Spring Boot: Framework para a criação do back-end e APIs RESTful.
## PostgreSQL: Sistema gerenciador de banco de dados objeto-relacional, escolhido pela praticidade nas consultas e segurança.
Arquitetura do Sistema
A arquitetura do sistema é dividida em camadas responsáveis pela troca de informações entre a interface do usuário e os dados no banco.

# Camadas e Pacotes - Back-End
## Config
Contém classes de configuração da aplicação, como configuração de segurança, banco de dados, CORS, etc.

## Controller
Responsável por receber as requisições HTTP do cliente, processar essas requisições e retornar as respostas apropriadas. Contém classes que atuam como controladores REST, mapeando endpoints para métodos específicos.

## Exception
Define as exceções personalizadas usadas na aplicação. Contém classes para diferentes tipos de erros que podem ocorrer durante a execução da aplicação.

## Handler
Contém classes que lidam com exceções e erros globais. Utilizado para capturar exceções de forma centralizada e retornar respostas consistentes para o cliente.

## Model
1. DTO (Data Transfer Object): Contém classes usadas para transferir dados entre as camadas da aplicação, especialmente entre o cliente e o servidor.
2. Entity: Define as entidades JPA que mapeiam tabelas do banco de dados.
3. Mapper: Contém classes que mapeiam entidades para DTOs e vice-versa, facilitando a conversão de dados.

## Repository
Contém interfaces que estendem JpaRepository ou outras interfaces do Spring Data JPA. Responsável pela comunicação com o banco de dados, permitindo operações CRUD e consultas personalizadas.

## Security
Contém classes relacionadas à segurança da aplicação, como configuração de autenticação e autorização. Inclui filtros de segurança, provedores de autenticação e outras implementações necessárias para proteger a aplicação.

## Service
1. Service: Define interfaces para os serviços que implementam a lógica de negócios da aplicação.
2. Service Impl: Implementa as interfaces de serviço, contendo a lógica de negócios real e chamando métodos do repositório conforme necessário.

## Shared
Constant: Contém constantes usadas em toda a aplicação, facilitando a manutenção e evitando a duplicação de valores estáticos.

## Utils
Contém classes utilitárias e funções auxiliares.

## Docker-Config
Inicialização dos Dockerfile.

## Tests
Contém os testes unitários do projeto.


---

## Kubernetes, Helm e GitOps (Argo CD)

Este projeto está pronto para ser implantado em Kubernetes usando Helm e GitOps com Argo CD.

### 1) Helm Chart
- Caminho: `helm/controle-estoque`
- Arquivos:
  - `Chart.yaml`
  - `values.yaml` (padrão)
  - `values-dev.yaml` e `values-prod.yaml`
  - `templates/` (Deployment, Service, Secret, Ingress opcional e Job de migrations)

O Job de migrations é executado antes do Deployment usando Argo CD Sync Waves e Hooks. As credenciais do banco estão nos `values-*.yaml` e são materializadas como `Secret` pelo Helm (apenas para fins didáticos desta etapa).

Variáveis de ambiente adicionais podem ser definidas em `values*.yaml` na chave `env:` e serão injetadas no Pod.

### 2) Modo de execução de migrations
A aplicação foi adaptada para suportar a variável `RUN_MIGRATIONS_ONLY=true`. Quando presente, ela inicia o contexto Spring, executa o Flyway e finaliza. O Job de migrations do Helm usa este modo automaticamente.

### 3) Build e publicação da imagem Docker (GHCR)
- Workflow: `.github/workflows/build-and-publish.yml`
- A imagem é publicada em `ghcr.io/<USUARIO>/<REPO>`.
- A tag preferencial é baseada no short SHA (`sha-<commit>`).
- O workflow, ao publicar uma nova imagem, atualiza as tags de imagem nos arquivos `values-dev.yaml` e `values-prod.yaml` e faz commit com `[skip ci]` para evitar loop.

Pré‑requisitos:
- Habilitar o GitHub Packages (GHCR) no repositório/organização.
- O `GITHUB_TOKEN` padrão já possui permissão `packages: write` via `permissions` no workflow.

### 4) Argo CD e GitOps
Este repositório contém uma estrutura de exemplo de um repositório GitOps em `argocd-gitops/` para facilitar a avaliação (na prática, crie outro repositório dedicado):
- `argocd-gitops/root-application.yaml`: App raiz (App of Apps) apontando para a pasta `apps`.
- `argocd-gitops/apps/operator-cloudnativepg.yaml`: instala o operador CloudNativePG (PostgreSQL) via Helm.
- `argocd-gitops/apps/db-dev.yaml` e `argocd-gitops/db/dev/cluster.yaml`: cluster Postgres para `dev`.
- `argocd-gitops/apps/db-prod.yaml` e `argocd-gitops/db/prod/cluster.yaml`: cluster Postgres para `prod`.
- `argocd-gitops/apps/app-dev.yaml` e `argocd-gitops/apps/app-prod.yaml`: aplicações apontando para este chart, carregando os respectivos `values-*.yaml`.

As anotações `argocd.argoproj.io/sync-wave` garantem a ordem: operador -> DB -> migrations -> aplicação.

### 5) Ajustes necessários
- Substitua `SEU_USUARIO` pelo seu usuário/organização GitHub em:
  - `helm/controle-estoque/values*.yaml`
  - `helm/controle-estoque/Chart.yaml` (campo `home`)
  - `argocd-gitops/**/*.yaml` (URLs de repo)
- Ajuste o host do Ingress em `values-prod.yaml`.

### 6) Comandos úteis
- Gerar e publicar imagem (via workflow) ao dar push em `main/master` ou manualmente via `workflow_dispatch`.
- Instalar em um cluster (exemplo ambiente dev) diretamente com Helm:
  ```bash
  # Namespace dev
  kubectl create ns dev || true
  helm upgrade --install ce-dev helm/controle-estoque -n dev -f helm/controle-estoque/values-dev.yaml
  ```
- Via Argo CD (recomendado): aplique o app raiz e acompanhe a sincronização:
  ```bash
  kubectl apply -n argocd -f argocd-gitops/root-application.yaml
  # Ou crie cada Application em argocd-gitops/apps/... conforme necessidade
  ```

### 7) Observações de segurança
- Não é recomendado manter credenciais em `values*.yaml` em produção. Use Secrets gerenciados (ex.: External Secrets, SOPS) e/ou gerenciadores (Vault, AWS Secrets Manager, etc.).
- Limite o acesso ao repositório GitOps e evite commits de segredos.

### 8) Operador de Banco (CloudNativePG)
- A instalação do operador é feita via Helm pelo Argo CD.
- Os serviços gerados pelo operador expõem endpoints `*-rw` (leitura/escrita). Os `values*.yaml` já apontam por padrão para `postgres-rw.<namespace>.svc.cluster.local` (ajuste conforme o nome real do cluster gerado, neste exemplo `ce-pg`).



## Variáveis de ambiente (.env)
Este projeto usa arquivos `.env` para configurar facilmente a aplicação localmente com Docker Compose.

Arquivos fornecidos:
- `.env` (padrão para dev) — já preenchido com valores locais comuns
- `.env.dev` — alternativa pronta para desenvolvimento
- `.env.example` — modelo com placeholders para você copiar e personalizar

Chaves disponíveis:
- APP_PORT: Porta exposta da API (padrão 8080)
- SPRING_PROFILES_ACTIVE: Perfil Spring (ex.: dev, prod)
- POSTGRES_PORT, POSTGRES_DB, POSTGRES_USER, POSTGRES_PASSWORD: parâmetros do Postgres
- JWT_SECRET: segredo do JWT para assinar tokens (use um valor forte em produção)
- JWT_EXPIRE_MS: tempo de expiração do JWT em milissegundos (padrão 3600000 = 1h)
- CORS_ORIGINS: lista de origens permitidas separadas por vírgula

Como usar com Docker Compose:
- Usando o `.env` padrão (na raiz):
  ```bash
  docker compose --env-file .env up --build
  ```
- Usando o `.env.dev`:
  ```bash
  docker compose --env-file .env.dev up --build
  ```

Observações técnicas:
- O `docker-compose.yml` injeta variáveis para o container da API, incluindo `SPRING_DATASOURCE_*`, `JWT_SECRET`, `CORS_ORIGINS` e `SPRING_PROFILES_ACTIVE`.
- O `application.yml` foi ajustado para ler as variáveis de ambiente e possui valores default seguros para desenvolvimento.
- Em Kubernetes/Helm as variáveis são gerenciadas via `values*.yaml` e `Secret`; os arquivos `.env` são apenas para ambiente local com Docker Compose.


## Implantação com Argo CD e Argo Rollouts (BlueGreen em Dev, Canary em Prod)
Este repositório já contém manifests GitOps para instalar o Argo Rollouts no cluster e configurar a aplicação para realizar deploys progressivos:

- Instalação do Argo Rollouts via Argo CD: `argocd-gitops/apps/argo-rollouts.yaml` (CRDs + controller + dashboard)
- Dev (namespace `dev`): estratégia BlueGreen com promoção manual (autoPromotion desabilitada)
- Prod (namespace `prod`): estratégia Canary com roteamento de tráfego via NGINX Ingress

### Pré-requisitos
- Argo CD instalado e acessível no namespace `argocd`.
- NGINX Ingress Controller instalado (para canary em produção).

### Bootstrapping com Argo CD
1. Aplique a aplicação raiz (App of Apps):
   ```bash
   kubectl apply -n argocd -f argocd-gitops/root-application.yaml
   ```
2. No Argo CD, sincronize a app raiz, o que criará:
   - `argocd-gitops/apps/argo-rollouts.yaml` (instala o Argo Rollouts)
   - `argocd-gitops/apps/app-dev.yaml` (aponta para este chart com `values-dev.yaml`)
   - `argocd-gitops/apps/app-prod.yaml` (aponta para este chart com `values-prod.yaml`)

### Como a chart Helm foi adaptada para Argo Rollouts
- Quando `rollout.enabled=true` (ver `values-*.yaml`), o `Deployment` padrão é suprimido e um recurso `Rollout` é criado (templates/rollout.yaml).
- Para BlueGreen (dev):
  - São criados services `-active` e `-preview` (templates/services-rollout.yaml).
  - O Ingress (se habilitado) aponta para `-active`.
  - `autoPromotionEnabled=false` por padrão em `values-dev.yaml`, exigindo promoção manual.
- Para Canary (prod):
  - São criados services `-stable` e `-canary` (templates/services-rollout.yaml).
  - O Ingress aponta para `-stable` e o Argo Rollouts controla os pesos via anotações do NGINX (stableIngress).
  - Exemplo de `steps` configurado em `values-prod.yaml`.
- O HPA (se habilitado) aponta para `Rollout` quando `rollout.enabled=true`.

### Instalando/Rodando o Dashboard do Argo Rollouts
O app `argo-rollouts` já habilita o Dashboard. Para acesso rápido via port-forward:
```bash
kubectl -n argo-rollouts port-forward svc/argo-rollouts-dashboard 3100:3100
```
Abra http://localhost:3100

### Fluxo BlueGreen em Dev: promoção manual
1. Faça uma alteração de versão (ou rode um deploy que mude a imagem) em `dev`.
2. Observe no Dashboard ou CLI que um novo ReplicaSet é criado e fica como `Preview`.
3. Teste o serviço `-preview` (se exposto).
4. Promova manualmente quando estiver tudo ok:
   ```bash
   # Verifique o nome do Rollout
   kubectl -n dev get rollouts

   # Promove a versão (switch de preview para active)
   kubectl -n dev argo rollouts promote <nome-do-rollout>
   ```
5. Se quiser abortar/reverter antes da promoção:
   ```bash
   kubectl -n dev argo rollouts abort <nome-do-rollout>
   ```

### Fluxo Canary em Prod: ajuste de tráfego
1. Em produção, o Ingress class é `nginx` e o Rollout está configurado com `trafficRouting.nginx`.
2. Durante a demo, é possível ajustar manualmente o peso do tráfego da canary:
   ```bash
   # Verifique o nome do Rollout
   kubectl -n prod get rollouts

   # Ajuste o peso para 20%
   kubectl -n prod argo rollouts set-weight <nome-do-rollout> 20

   # Ajuste para 50%
   kubectl -n prod argo rollouts set-weight <nome-do-rollout> 50

   # Retornar o tráfego para 0% ou finalizar a promoção
   kubectl -n prod argo rollouts set-weight <nome-do-rollout> 0
   # ou
   kubectl -n prod argo rollouts promote <nome-do-rollout>
   ```
3. Também é possível pausar e retomar:
   ```bash
   kubectl -n prod argo rollouts pause <nome-do-rollout>
   kubectl -n prod argo rollouts resume <nome-do-rollout>
   ```

### Notas importantes
- Certifique-se de ter o plugin/CLI `kubectl-argo-rollouts` instalado para usar os comandos `kubectl argo rollouts ...`.
- Para Canary com NGINX, o nome do Ingress está definido pelo chart como `{{ include "controle-estoque.fullname" . }}`; você pode sobrepor em `values-prod.yaml` via `rollout.canary.trafficRouting.nginx.stableIngress` caso necessário.
- Se você não deseja usar Argo Rollouts em algum ambiente, defina `rollout.enabled=false` e o chart voltará a usar `Deployment` e `Service` únicos.
