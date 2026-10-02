# Paperclip — configuração dos agentes

Como o Paperclip foi configurado para que um agente de IA receba este repositório, trabalhe em uma branch isolada a partir de `develop` e entregue por Pull Request.

Este guia registra apenas o que foi testado e funcionou.

## Onde paramos

- Paperclip configurado e validado: a tarefa **CEL-2** rodou em worktree isolada a partir de `develop` (seção 5).
- A tarefa **CEL-3 — recuperação de senha** (seção 6) foi executada pelo agente, que abriu o **PR #1**. A revisão encontrou problemas (seção 6, _Revisão do PR #1_); as lições foram incorporadas às instruções dos agentes do tutorial.
- A CI (seção 7) foi criada em `.github/workflows/ci.yml`, mas ainda não está na `develop`.
- **Decisão:** a recuperação de senha será refeita do zero para o tutorial, a partir de `develop` em `b0158f4`, com o fluxo Analista → Backend Developer → Responsável → homologação.
- **Próximo passo:** seguir a preparação em [PAPERCLIP-TUTORIAL.md](PAPERCLIP-TUTORIAL.md) (fechar o PR #1, convidar pessoas, criar e ajustar os agentes, decidir sobre a CI).

### Contexto para continuar com o Claude Code em outro computador

A memória do Claude Code fica no computador onde foi gravada. No novo computador, comece a conversa pedindo para ler este arquivo, por exemplo: _"Leia docs/PAPERCLIP.md do backend e continue a partir de 'Onde paramos'"_. Decisões que valem para a continuação:

- Projetos locais: `C:\celke\gestao-comercial-precificacao-backend` e `C:\celke\gestao-comercial-precificacao-frontend`; repositórios `celkecursos/gestao-comercial-precificacao-backend` e `celkecursos/gestao-comercial-precificacao-frontend`.
- Os prompts de origem ficam em `C:\celke\prompt-gerar-backend.md` e `C:\celke\prompt-gerar-frontend.md`. Ao mudar requisitos, atualize-os como se a orientação sempre tivesse existido, sem marcar data ou alteração.
- A **recuperação de senha não deve ser implementada manualmente**: ela é a demonstração do fluxo com o agente do Paperclip. O Claude apenas ajuda a configurar, acompanhar e revisar.
- Commits sem o trailer `Co-Authored-By` do Claude.
- Stack mantida em NestJS 11 (Jest/ESLint); não atualizar para o Nest 12 sem perguntar.
- MySQL local de desenvolvimento via `docker compose up -d mysql`, porta **3307**.

## Visão geral

```text
GitHub (develop)
   │
   ▼
Paperclip clona o repositório na pasta gerenciada do projeto
   │
   ▼
Para cada tarefa: git worktree própria + branch <ID>-<slug> a partir de origin/develop
   │
   ▼
Agente (Codex) trabalha na worktree
   │
   ▼
Pull Request ──► CI (GitHub Actions) ──► revisão humana ──► develop ──► homolog ──► main
```

| Branch    | Ambiente | Uso                                             |
| --------- | -------- | ----------------------------------------------- |
| `develop` | Develop  | Integração; base de todas as branches de tarefa |
| `homolog` | Homolog  | Testes humanos                                  |
| `main`    | Produção | Produção                                        |

Os agentes nunca alteram `develop`, `homolog` ou `main` diretamente.

## Pré-requisitos

- Repositório no GitHub: `celkecursos/gestao-comercial-precificacao-backend`, com as branches `main`, `homolog` e `develop`.
- No Paperclip, o GitHub conectado e o app **Paperclip for GitHub** com acesso ao repositório.
- Uma chave `OPENAI_API_KEY` para o agente Codex.
- A CI do repositório (seção 7). O Paperclip roda na VPS, e o ambiente do agente não tem MySQL: o agente não consegue rodar `test:e2e` nem conferir migrations, e quem faz essa verificação é a CI.

## 1. Habilitar workspaces isolados na instância

**Settings** (engrenagem no rodapé da barra lateral) → **Instance settings** → **Experimental** → ligar **Isolated workspaces**.

Sem essa opção, a política de workspace do projeto é ignorada e o agente roda em uma pasta vazia, sem `.git`, com a mensagem `No project or prior session workspace was available. Using fallback workspace`.

Se a opção não aparecer na tela, ligue pelo console do navegador (F12), logado no Paperclip:

```js
await fetch('/api/instance/settings/experimental', {
  method: 'PATCH',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ enableIsolatedWorkspaces: true }),
}).then((r) => r.json());
```

## 2. Criar e configurar o projeto

**Projects → New project**, com o nome `gestao-comercial-precificacao-backend` e, em _Source repos_, o repositório `celkecursos/gestao-comercial-precificacao-backend` (GitHub).

### Workspace do repositório

**Projeto → Workspaces → celkecursos/gestao-comercial-precificacao-backend**:

| Campo                                  | Valor                                                                               |
| -------------------------------------- | ----------------------------------------------------------------------------------- |
| Source type                            | Remote git repo                                                                     |
| Repo URL                               | `https://github.com/celkecursos/gestao-comercial-precificacao-backend` (sem `.git`) |
| Repo ref                               | `develop`                                                                           |
| Default ref                            | `develop`                                                                           |
| Local path                             | vazio (pasta gerenciada pelo Paperclip)                                             |
| Setup / Cleanup command                | vazios                                                                              |
| Shared workspace key / Remote provider | vazios                                                                              |

Os textos em cinza nesses campos (`origin/main`, `pnpm install && pnpm dev`, `codespaces`...) são apenas exemplos, não valores preenchidos.

### Execution Workspaces

**Projeto → Configuration → Execution Workspaces**:

| Opção                                  | Valor  |
| -------------------------------------- | ------ |
| Enable isolated task checkouts         | ligado |
| New tasks default to isolated checkout | ligado |
| Shared workspace concurrency           | Auto   |

Em **Show advanced checkout settings**:

| Campo                                    | Valor                                                                                      |
| ---------------------------------------- | ------------------------------------------------------------------------------------------ |
| Implementation                           | Git worktree                                                                               |
| Base ref                                 | `develop`                                                                                  |
| Branch template                          | vazio (padrão `{{issue.identifier}}-{{slug}}`)                                             |
| Worktree parent dir                      | vazio (padrão `.paperclip/worktrees`)                                                      |
| Provision / Runtime provision / Teardown | **vazios** — o repositório não tem a pasta `scripts/`; preenchê-los faz toda tarefa falhar |

### Conferência

Abra `https://<seu-paperclip>/api/projects/<id-do-projeto>` e confira:

```json
"executionWorkspacePolicy": {
  "enabled": true,
  "defaultMode": "isolated_workspace",
  "allowIssueOverride": true,
  "workspaceStrategy": { "type": "git_worktree", "baseRef": "develop", "branchTemplate": "", "worktreeParentDir": "" }
},
"codebase": { "repoRef": "develop", "defaultRef": "develop", "origin": "managed_checkout" }
```

O ID do projeto aparece no caminho da pasta gerenciada: `/data/instances/default/projects/<id-da-empresa>/<id-do-projeto>/...`.

## 3. Criar o agente Backend Developer

**Agents → New agent**:

| Etapa     | Campo           | Valor                                                       |
| --------- | --------------- | ----------------------------------------------------------- |
| Connect   | Runtime         | Codex, com a `OPENAI_API_KEY`                               |
| Configure | Model           | Default                                                     |
| Configure | Thinking effort | Auto                                                        |
| Configure | Environment     | **Local** — não trocar: é onde ficam o clone e as worktrees |

Rode **Run test** e conclua com **Finish setup**. No teste do agente, a mensagem "fallback workspace" é esperada, porque o teste não está ligado a nenhuma tarefa.

Em **Edit configuration**, defina o _Title_ `Backend Developer — Gestão Comercial e Precificação` e o _Reports to_ (o agente principal da empresa).

### Instruções do agente

Em **Agents → Backend Developer → Instructions → AGENTS.md** (botão **edit**; **raw** mostra o texto salvo), mantenha o contrato padrão do Paperclip que já vem no arquivo e acrescente ao final o bloco abaixo, uma única vez. Para mudar as instruções depois, edite o mesmo bloco nessa tela: o AGENTS.md do repositório não atualiza o do Paperclip.

```markdown
## Project: Gestão Comercial e Precificação — Backend

Você é o desenvolvedor backend deste projeto (Node.js 24, TypeScript, NestJS 11, TypeORM,
MySQL 8, JWT, Jest, Swagger). Responda e escreva comentários em português.

### Antes de qualquer alteração

- No repositório do projeto, leia AGENTS.md, README.md, docs/ARCHITECTURE.md e docs/ROADMAP.md
  e siga as convenções deles. (Este arquivo de instruções é do Paperclip; o AGENTS.md do
  repositório é outro e também deve ser seguido.)
- Confirme a branch com `git branch --show-current`. Você deve estar na branch própria da
  tarefa, criada a partir de develop. Se estiver em main, homolog ou develop, NÃO altere nada:
  marque a tarefa como blocked explicando o motivo.

### Regras de Git

- Nunca faça commit ou push em main, homolog ou develop.
- Entregue sempre por Pull Request da branch da tarefa para develop, e registre o PR como
  work product (pull_request). Nunca faça merge.

### Escopo

- Faça apenas o que a tarefa pede. Em tarefa marcada como READ-ONLY, não altere arquivos,
  não instale dependências, não faça commit, push nem PR: reporte o resultado em comentário
  e marque a tarefa como done.
- Nunca exponha segredos nem versione .env.
- Nova variável de ambiente: declare em src/config/env.validation.ts e no .env.example.
- Alterou entidades: gere migration e confirme que
  `npm run typeorm -- migration:generate src/database/migrations/Check --dryrun`
  não aponta pendências.

### Definição de pronto (tarefas de implementação)

npm run lint, npm run test, npm run test:e2e e npm run build passando.
Se algum comando não puder rodar no ambiente (ex.: sem MySQL para o e2e), diga isso
explicitamente. Nunca afirme que passou sem ter executado.

O repositório tem CI (GitHub Actions) que roda todos esses comandos com MySQL em cada PR.
Depois de abrir ou atualizar o PR, acompanhe com `gh pr checks <número> --watch`.
CI vermelha: leia `gh run view <id> --log-failed`, corrija na mesma branch e envie de novo.
Só marque a tarefa como concluída com a CI verde.

### Relatório final (comentário na tarefa)

Branch, arquivos alterados, comandos executados com resultado, status da CI, link do PR e limitações.
```

## 4. Como criar tarefas

Crie as tarefas **de dentro do projeto** (**Projects → gestao-comercial-precificacao-backend → Tasks → New task**), e não pelo botão _Assign Task_ do agente.

No formulário, `For Assignee` e `in Project` são apenas textos de exemplo: escolha explicitamente o responsável e confira o nome do projeto. Uma tarefa sem projeto roda fora do repositório.

| Campo               | Valor                                 |
| ------------------- | ------------------------------------- |
| For                 | Backend Developer                     |
| in                  | gestao-comercial-precificacao-backend |
| Execution workspace | New isolated workspace                |

Depois de criar, confira em `https://<seu-paperclip>/api/issues/<ID>` que `projectId` está preenchido.

A tarefa `CEL-1` (_Paperclip onboarding_) é criada automaticamente na instalação e pertence ao agente principal; não faz parte deste fluxo.

## 5. Tarefa de verificação — CEL-2 (concluída)

Tarefa somente leitura usada para provar que o agente recebe o repositório.

**Título:** `Verificar workspace do backend (READ-ONLY)`

```text
Tarefa READ-ONLY. Não modifique arquivos, não instale nada, não faça commit, push nem PR.
Execute e reporte a saída literal de cada comando:

pwd
git rev-parse --show-toplevel
git branch --show-current
git remote get-url origin
git log -1 --oneline
git fetch origin develop && git merge-base --is-ancestor origin/develop HEAD && echo BASE_OK
ls package.json src README.md AGENTS.md
git status --porcelain

Ao final, comente o resultado na tarefa e marque como done.
```

Resultado obtido (`/api/issues/CEL-2`):

| Campo                                    | Valor                                                                                               |
| ---------------------------------------- | --------------------------------------------------------------------------------------------------- |
| `status`                                 | `done` (cerca de 1 minuto)                                                                          |
| `executionWorkspaceSettings`             | `{ "mode": "isolated_workspace" }`                                                                  |
| `currentExecutionWorkspace.strategyType` | `git_worktree`                                                                                      |
| `currentExecutionWorkspace.baseRef`      | `origin/develop`                                                                                    |
| `currentExecutionWorkspace.branchName`   | `CEL-2-verificar-workspace-do-backend-read-only`                                                    |
| `currentExecutionWorkspace.cwd`          | `<pasta gerenciada do projeto>/.paperclip/worktrees/CEL-2-verificar-workspace-do-backend-read-only` |
| `deliveryState`                          | `merged_by_ancestry` — esperado: sem commits, a branch é igual à `develop`                          |

## 6. Tarefa CEL-3 — recuperação de senha (em revisão)

Branch: `CEL-3-implementar-recupera-o-de-senha-por-e-mail-api`, criada pelo Paperclip a partir de `origin/develop`. O Paperclip descarta as letras acentuadas ao gerar o nome (`recuperação` → `recupera-o`).

O editor de tarefas do Paperclip converte a descrição para Markdown e escapa alguns caracteres (`PASSWORD\_RESET...`, `&#x20;`). Confira no PR se os nomes de variáveis saíram corretos.

**Título:** `Implementar recuperação de senha por e-mail (API)`

```text
Implemente a recuperação de senha por e-mail na API, conforme a seção
"Recuperação de senha por e-mail" do docs/ROADMAP.md.

Requisitos:
1. POST /auth/forgot-password (@Public) recebe { email }. Responde sempre 200 com a mesma
   mensagem genérica, exista ou não o e-mail (não revelar quais e-mails estão cadastrados).
2. Se o usuário existir e estiver ativo, gere um token aleatório seguro, grave apenas o HASH
   do token (entidade + migration), com expiração (variável PASSWORD_RESET_TOKEN_TTL_MINUTES,
   padrão 30) e uso único. Um novo pedido invalida os tokens anteriores do usuário.
3. Envie e-mail com o link {FRONTEND_RESET_PASSWORD_URL}?token=<token>.
   - Serviço de e-mail configurável por variáveis SMTP (host, porta, usuário, senha, remetente),
     declaradas em env.validation.ts e no .env.example.
   - Sem SMTP configurado e fora de produção: registre o link no log em vez de enviar.
     Em produção, SMTP é obrigatório.
4. POST /auth/reset-password (@Public) recebe { token, newPassword }:
   - valide newPassword com @IsStrongPassword();
   - token inválido, expirado ou já usado → 400 com mensagem clara;
   - grave a senha SOMENTE via UsersService.updatePassword() (que já invalida os JWTs antigos);
   - marque o token como usado.
5. Limite de tentativas no forgot-password para evitar abuso (por IP e/ou e-mail).
6. Swagger completo nas duas rotas (@ApiOperation, respostas e @ApiErrorResponses).
7. Testes unitários do service e e2e do fluxo completo: pedido, e-mail/log, redefinição,
   token reutilizado, token expirado, senha fraca, e-mail inexistente.
8. Atualize o README (seção "Recuperação de senha", variáveis de ambiente, endpoints) e o
   docs/ROADMAP.md (marcar como implementada).

Fora do escopo: telas do frontend (haverá tarefa separada no projeto frontend).

Git: trabalhe na branch da tarefa (a partir de develop), commits descritivos, abra Pull Request
para develop. Não faça merge.

Definição de pronto: lint, test, test:e2e, build e o migration:generate --dryrun sem pendências.
No PR e no relatório, liste: endpoints, variáveis novas, migration criada e resultado de cada comando.
```

### Revisão do PR #1

O agente entregou lint, testes unitários e build passando, mas não rodou `test:e2e` nem a conferência de migrations (sem MySQL no ambiente). Rodando com MySQL:

| Verificação                   | Resultado                                                                    |
| ----------------------------- | ---------------------------------------------------------------------------- |
| `test:e2e`                    | Falha em `GET /dashboard`: o teste de recuperação cria um usuário (5, não 4) |
| `migration:generate --dryrun` | Pendência: nome da FK escrito à mão difere do gerado pelo TypeORM            |

Comentário para a tarefa CEL-3:

```text
Não aprovado ainda. Rodei test:e2e e migration:generate --dryrun com MySQL:

1. test:e2e falha em "GET /dashboard retorna os quatro cards": o usuário criado no teste de
   recuperação faz users ser 5, não 4. Ajuste a expectativa/comentário.
2. migration:generate --dryrun aponta pendência: recriar a FK com nome gerado. Adicione
   foreignKeyConstraintName: 'FK_password_reset_tokens_user_id' no @JoinColumn da entidade.
3. resetPassword: torne o uso do token atômico — update condicional (usedAt IS NULL) e só chame
   updatePassword se affected === 1.
4. Rate limit: remova do Map as chaves sem tentativas na janela (o Map hoje cresce sem limite).
5. forgotPassword: não deixe falha de SMTP virar 500 (revela e-mail cadastrado); registre no log.
6. Restaure os comentários e linhas em branco removidos do AuthService (fora do escopo).

Atualize o PR #1 na mesma branch. Não consigo rodar e2e no seu ambiente; eu reexecuto aqui.
```

Com a CI ativa, a última linha pode ser trocada por: _"Atualize o PR #1 na mesma branch e acompanhe a CI até ficar verde."_

### Depois do Pull Request

1. Conferir a CI verde e revisar o diff no PR; fazer o merge em `develop` → deploy automático do ambiente Develop.
2. PR `develop → homolog` → testes humanos no ambiente Homolog.
3. PR `homolog → main` → produção.
4. Cadastrar na Hostinger, em cada ambiente, as novas variáveis (SMTP e URL de redefinição de senha).
5. Criar o agente **Frontend Developer** no projeto do frontend, seguindo este mesmo guia, para as telas de recuperação de senha.

## 7. CI — GitHub Actions

Arquivo: `.github/workflows/ci.yml`. Roda em todo Pull Request para `develop`, `homolog` e `main` e a cada push nessas branches, com um serviço MySQL 8.4:

`npm ci` → `lint` → `test` → `build` → `migration:run` → `migration:generate --check` → `test:e2e`

O `--check` falha quando as entidades divergem das migrations (o `--dryrun` só mostra a diferença e termina com sucesso). As variáveis ficam no próprio workflow; não há `.env` nem segredos na CI.

Com a CI, a verificação não depende da máquina de ninguém: o agente acompanha o resultado pelo `gh` e o revisor vê o status no PR.

### Exigir a CI antes do merge

No GitHub: **Settings → Branches** (ou **Rules**) → regra para `develop` → **Require status checks to pass** → marcar **Lint, testes, build e migrations**. Repetir para `homolog` e `main`.

O check só aparece na lista depois de rodar pelo menos uma vez.

## Problemas encontrados e soluções

| Sintoma                                                              | Causa                                                                                                  | Solução                                                                                               |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------- |
| `no Codex credentials available for managed home`                    | Agente Codex sem credencial                                                                            | Vincular a `OPENAI_API_KEY` ao agente                                                                 |
| `Using fallback workspace` e pasta sem `.git` em uma tarefa          | Isolated workspaces desligado na instância, projeto sem política de execução ou tarefa sem `projectId` | Seções 1, 2 e 4                                                                                       |
| A pasta gerenciada `/data/instances/default/projects/...` não existe | O Paperclip só clona o repositório na primeira tarefa que usa o workspace do projeto                   | Nenhuma: é criada automaticamente                                                                     |
| "checked out" no histórico da tarefa sem código disponível           | No Paperclip, _checkout_ significa que o agente assumiu a tarefa, não `git checkout`                   | —                                                                                                     |
| Agente não roda `test:e2e` nem confere migrations                    | O ambiente do Paperclip na VPS não tem MySQL nem Docker                                                | CI no GitHub Actions (seção 7)                                                                        |
| Deploy da API executando o build do frontend                         | URLs dos repositórios trocadas no `git remote`                                                         | `git remote set-url origin <url correta>` e `git push --force-with-lease origin main homolog develop` |
