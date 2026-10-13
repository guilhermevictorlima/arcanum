# Fase 0 — Plano de execução (Claude Code)

Este plano leva o projeto recém-gerado até uma base segura e verde no CI, em 10 passos pequenos, cada um com verificação.

## Regras para o Claude Code

- Ler o `pom.xml` antes de tudo e descobrir o `groupId` e o pacote base; usar `<pacote-base>` nos exemplos abaixo.
- Fixar versões de imagens Docker e de GitHub Actions (por tag exata; Actions por SHA de commit).
- Todas as portas do Compose publicadas só em `127.0.0.1`.
- Nada de segredo em `application*.yml`, Compose ou scripts: apenas variáveis `${...}` vindas do `.env`.
- Se um passo exigir decisão não prevista aqui, parar e perguntar.

## Pré-requisitos na máquina

- [ ] JDK compatível com o `pom.xml`, Docker com Compose v2, Git
- [ ] `mkcert` instalado e a CA local registrada (`mkcert -install`)
- [ ] `pre-commit` instalado (`pipx install pre-commit` ou gerenciador do sistema)
- [ ] Repositório Git inicializado, com remoto no GitHub

## Passo 0.1 — Higiene do repositório

Objetivo: impedir, desde o primeiro commit, que segredo ou artefato local entre no Git.

- [ ] `.gitignore` cobrindo `.env`, `.local/`, `*.p12`, `*.pem`, `*.key`, `target/`, `.idea/` e `*.iml`
- [ ] `.env.example` com todas as variáveis esperadas e valores de mentira (`change-me`)
- [ ] `.gitattributes` com `* text=auto eol=lf`
- [ ] `README.md` inicial: o que é o Arcanum, aviso de que é projeto de estudos e link para este plano
- [ ] `SECURITY.md` curto explicando como reportar uma falha

Verificação: `git status` limpo após criar um `.env` de teste (ele não pode aparecer como arquivo novo).

## Passo 0.2 — Maven endurecido

Objetivo: build reproduzível, com cadeia de suprimentos verificada.

- [ ] Maven Wrapper presente e `distributionSha256Sum` definido em `.mvn/wrapper/maven-wrapper.properties`
- [ ] `.mvn/maven.config` com `--strict-checksums` e `--batch-mode`
- [ ] `maven-enforcer-plugin` na fase `validate` com `requireMavenVersion`, `requireJavaVersion`, `dependencyConvergence`, `banDuplicatePomDependencyVersions` e `requireUpperBoundDeps`
- [ ] Dependências adicionadas pelo Initializr ou à mão: Web, Security, Validation, Data JPA, driver PostgreSQL, Flyway (`flyway-core` e `flyway-database-postgresql`), Actuator; de teste: Spring Boot Test, Spring Security Test, Testcontainers (PostgreSQL e JUnit 5) e ArchUnit
- [ ] Redis, JWT e springdoc ficam para as fases em que forem usados

Verificação: `./mvnw -q validate` passa; adicionar uma dependência duplicada de propósito faz o Enforcer falhar (desfazer depois).

## Passo 0.3 — Estrutura de pacotes e ArchUnit

Objetivo: travar a arquitetura antes que o código cresça.

- [ ] Pacotes vazios com `package-info.java` em `<pacote-base>`: `auth`, `account`, `vault`, `security`, `audit`, `common`
- [ ] Cada domínio com subpacotes `web` (controllers e DTOs), `service`, `persistence`
- [ ] Teste `ArchitectureTest` com regras: controllers não acessam repositórios; sem dependência cíclica entre domínios; sem injeção por campo (`@Autowired` em atributo); `persistence` não depende de `web`

Verificação: `./mvnw test -Dtest=ArchitectureTest` verde.

## Passo 0.4 — Ambiente local com Docker Compose

Objetivo: toda a infraestrutura sobe com um comando, de forma parecida com produção.

- [ ] `compose.yaml` com PostgreSQL, Redis (com senha), Vault em modo dev e Mailpit; imagens com tag fixa, `127.0.0.1` nas portas, `healthcheck` e `security_opt: [no-new-privileges:true]` em todos
- [ ] Script `scripts/setup-local.sh` que gera valores aleatórios (`openssl rand`) e grava o `.env`, sem sobrescrever um existente
- [ ] Script `docker/postgres/init/01-roles.sh` criando dois papéis: `arcanum_migrator` (dono do schema, executa DDL) e `arcanum_app` (só `SELECT`, `INSERT`, `UPDATE`, `DELETE`); revogar `CREATE` no schema `public` de `PUBLIC`
- [ ] Script `scripts/seed-vault.sh` gravando valores de exemplo em `secret/arcanum` no Vault dev (uso real só na Fase 3)
- [ ] README com o aviso de que o Vault em modo dev é em memória e perde tudo ao reiniciar

Verificação: `./scripts/setup-local.sh && docker compose up -d` deixa todos os serviços `healthy`.

## Passo 0.5 — TLS local (aplicação e PostgreSQL)

Objetivo: nenhuma conexão em texto puro, nem em desenvolvimento.

- [ ] `scripts/setup-certs.sh` gerando com `mkcert` o certificado de `localhost`, `127.0.0.1` e `::1` em `.local/certs/` (PEM para o Postgres, PKCS12 para o Spring)
- [ ] PostgreSQL com `ssl=on`; atenção: a chave precisa pertencer ao usuário `postgres` com modo `0600`, então o container deve copiar os certificados montados para um caminho próprio e ajustar dono e permissão antes de iniciar
- [ ] `pg_hba.conf` aceitando apenas `hostssl` para conexões TCP
- [ ] JDBC com `sslmode=verify-full` e `sslrootcert` apontando para a CA do mkcert
- [ ] Spring Boot com `server.ssl.*` lendo o PKCS12 e a senha de variáveis de ambiente, porta 8443

Verificação: `psql "sslmode=verify-full ..."` conecta; `psql "sslmode=disable ..."` é recusado.

## Passo 0.6 — Configuração da aplicação

Objetivo: configuração por perfil, sem segredo e com defaults seguros.

- [ ] `application.yml` base e `application-dev.yml`, `application-test.yml`; sem senha literal em nenhum deles
- [ ] Datasource com o usuário `arcanum_app`; Flyway com o usuário `arcanum_migrator` (`spring.flyway.user` e `password` separados)
- [ ] `spring.jpa.hibernate.ddl-auto=validate`, `spring.jpa.open-in-view=false`, `spring.flyway.fail-on-missing-locations=true`
- [ ] `server.error.include-stacktrace=never`, `include-message=never`, `include-binding-errors=never`; `server.max-http-request-header-size` e limite de corpo de requisição conservadores
- [ ] Actuator expondo somente `health` e `info`, em porta de gerenciamento separada (8081), com `management.server.address=127.0.0.1` e o mesmo TLS
- [ ] `V1__baseline.sql` em `db/migration`: define `ALTER DEFAULT PRIVILEGES` para que tabelas futuras do migrator já deem DML ao `arcanum_app`

Verificação: a aplicação sobe com `./mvnw spring-boot:run -Dspring-boot.run.profiles=dev`; o Flyway aplica a V1 e a tabela `flyway_schema_history` existe.

## Passo 0.7 — Spring Security restritivo

Objetivo: a porta de entrada já nasce fechada.

- [ ] `SecurityConfig` em `<pacote-base>.security` com um `SecurityFilterChain`: sessão `STATELESS`, `anyRequest().denyAll()` exceto `GET /api/v1/ping` (endpoint temporário de teste, a remover na Fase 3)
- [ ] CSRF desligado, justificado em ADR: API stateless com bearer token e sem cookies de sessão
- [ ] Cabeçalhos: HSTS (incluindo subdomínios), `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer`, CSP `default-src 'none'; frame-ancestors 'none'`, `Cache-Control: no-store` em todas as respostas
- [ ] Excluir `UserDetailsServiceAutoConfiguration` para que o Spring não gere senha aleatória no log
- [ ] `PingController` retornando `{"status":"ok"}`
- [ ] Testes com MockMvc: `ping` responde 200 com todos os cabeçalhos acima; qualquer outra rota responde 401 ou 403; um `POST` sem autenticação não chega a nenhum handler

Verificação: `curl --cacert "$(mkcert -CAROOT)/rootCA.pem" -i https://localhost:8443/api/v1/ping` mostra os cabeçalhos; `curl -k http://localhost:8443` falha.

## Passo 0.8 — Testes de integração base

Objetivo: provar que a infraestrutura de segurança funciona, não só que compila.

- [ ] Classe base com Testcontainers PostgreSQL (`@ServiceConnection`), reaproveitando o script de papéis do passo 0.4
- [ ] `ApplicationContextIT`: contexto sobe e o Flyway aplica todas as migrações
- [ ] `DatabasePrivilegesIT`: conectado como `arcanum_app`, `CREATE TABLE` falha; conectado como `arcanum_migrator`, funciona
- [ ] `LogRedactionTest` esqueleto: garante que nenhum log de teste contém a palavra `password=` com valor (será ampliado na Fase 3)
- [ ] Plugin Failsafe rodando os `*IT` na fase `verify`

Verificação: `./mvnw verify` verde, sem Docker Compose rodando (apenas Testcontainers).

## Passo 0.9 — Qualidade e CI

Objetivo: toda mudança passa por varreduras automáticas.

- [ ] `.pre-commit-config.yaml` com o hook do `gitleaks`; `pre-commit install` documentado no README
- [ ] SpotBugs com `find-sec-bugs` configurado no Maven, rodando em `verify` e falhando em achados de severidade média ou alta
- [ ] Profile Maven `security` com `dependency-check-maven` (exige chave da API do NVD, vinda da variável `NVD_API_KEY`, nunca do `pom.xml`)
- [ ] `.github/workflows/ci.yml` com `permissions: contents: read`, jobs de build e testes (`./mvnw verify`), gitleaks, CodeQL (Java), Semgrep e dependency-check; todas as Actions fixadas por SHA
- [ ] `.github/dependabot.yml` para `maven`, `github-actions` e `docker`, semanal
- [ ] Proteção da branch `main` (PR obrigatório e CI verde) documentada no README, já que isso se configura no GitHub e não no código

Verificação: abrir um PR de teste com um segredo falso (`AKIA...` de exemplo) e confirmar que gitleaks barra; depois descartar o PR.

## Passo 0.10 — ADRs e fechamento

Objetivo: registrar as decisões enquanto elas estão frescas.

- [ ] `docs/adr/0001-stack.md`: Java, Spring Boot, Maven, PostgreSQL, Flyway e por que cada um
- [ ] `docs/adr/0002-segredos-locais.md`: `.env` gerado, Vault dev só em memória, quando o Vault entra na aplicação
- [ ] `docs/adr/0003-csrf-desabilitado.md`: justificativa do passo 0.7
- [ ] `docs/adr/0004-papeis-do-banco.md`: separação `arcanum_migrator` e `arcanum_app`
- [ ] README final com o passo a passo: `setup-certs.sh`, `setup-local.sh`, `docker compose up -d`, `./mvnw spring-boot:run`

## Critério de pronto da Fase 0

- [ ] `docker compose up -d` deixa PostgreSQL, Redis, Vault e Mailpit saudáveis
- [ ] A aplicação sobe em HTTPS (8443) e fala com o PostgreSQL com `verify-full`
- [ ] `./mvnw verify` verde, incluindo ArchUnit e testes de privilégio do banco
- [ ] CI verde no GitHub: build, gitleaks, CodeQL, Semgrep e dependency-check
- [ ] Nenhum segredo no histórico do Git (`gitleaks detect` limpo)
- [ ] Todas as rotas, exceto o `ping` temporário, respondem 401 ou 403
