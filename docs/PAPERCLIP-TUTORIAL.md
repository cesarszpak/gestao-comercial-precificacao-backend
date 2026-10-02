# Paperclip — preparação do tutorial

Preparação para gravar o tutorial do fluxo com o Paperclip, a partir do projeto base (`develop` em `b0158f4`). A configuração da instância, do projeto e do agente está em [PAPERCLIP.md](PAPERCLIP.md).

```text
Analista            Backend Developer (agente)     Responsável                    QA Homolog (agente) + Analista
cria a tarefa  ──▶  desenvolve e abre PR      ──▶  revisa, merge em develop  ──▶  testam em homologação
                    (status in_review)             promove para homolog           ──▶ Responsável promove para main
```

| Papel                    | Quem                     | Papel na organização do Paperclip |
| ------------------------ | ------------------------ | --------------------------------- |
| Analista                 | Pessoa                   | Operator                          |
| Responsável              | Pessoa                   | Operator                          |
| Backend Developer        | Agente (Codex)           | —                                 |
| QA Homolog               | Agente (Codex)           | —                                 |
| Revisor de código        | Agente (Codex), opcional | —                                 |
| Dono da instância        | Pessoa                   | Owner                             |

Os textos das tarefas (Analista, QA, teste de aceite) e os modelos de comentário do Responsável **não ficam no repositório**, para o agente desenvolvedor não ler as tarefas futuras nem o roteiro de testes. Eles estão no arquivo `ROTEIRO-PROMPTS.md`, mantido fora do repositório; copie-o para o outro computador.

## 1. Voltar ao zero

- [ ] GitHub: fechar o PR #1 sem merge e apagar a branch `CEL-3-implementar-recupera-o-de-senha-por-e-mail-api`. Use uma conta com permissão de escrita no repositório (a conta `cesarszpak` só tem leitura).
- [ ] GitHub: conferir que `develop` está em `b0158f4` (`git ls-remote origin develop`).
- [ ] Paperclip: cancelar a tarefa CEL-3. A numeração continua (a próxima será CEL-4). Para gravar a partir de CEL-1, crie uma empresa nova no Paperclip e refaça as seções 1 a 3 do [PAPERCLIP.md](PAPERCLIP.md).
- [ ] Hostinger: nada a fazer — o PR #1 nunca chegou à `develop`.

## 2. Pessoas

- [ ] **Settings → Organization Members → Invites**: gerar um convite **Operator** para o Analista e outro para o Responsável (cada link é de uso único).

## 3. Agentes

- [ ] **Backend Developer**: em **Instructions → AGENTS.md**, manter o contrato padrão do Paperclip e substituir a seção do projeto pelo texto de [3.1](#31-backend-developer).
- [ ] **QA Homolog**: criar como o Backend Developer (Codex, ambiente **Local**), com as instruções de [3.2](#32-qa-homolog).
- [ ] **Revisor de código** (opcional): criar da mesma forma, com as instruções de [3.3](#33-revisor-de-código-opcional).

### 3.1 Backend Developer

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
- Leia a tarefa inteira. A tarefa é escrita por um analista em linguagem de negócio: traduza
  as regras para a arquitetura do projeto. Se uma regra estiver ambígua ou conflitar com o
  código, não adivinhe: comente as perguntas na tarefa e marque como blocked.

### Regras de Git

- Nunca faça commit ou push em main, homolog ou develop.
- Entregue sempre por Pull Request da branch da tarefa para develop, e registre o PR como
  work product (pull_request). Nunca faça merge.
- Commits pequenos e descritivos, em português.

### Escopo

- Faça apenas o que a tarefa pede. Não reformate nem remova comentários de código fora do
  escopo.
- Em tarefa marcada como READ-ONLY, não altere arquivos, não instale dependências, não faça
  commit, push nem PR: reporte o resultado em comentário e marque a tarefa como done.
- Nunca exponha segredos nem versione .env.
- Nova variável de ambiente: declare em src/config/env.validation.ts e no .env.example.
- Alterou entidades: gere a migration com `npm run migration:generate` (não escreva nomes de
  constraints à mão) e confirme que
  `npm run typeorm -- migration:generate src/database/migrations/Check --dryrun`
  não aponta pendências.
- Mudou a contagem de registros criados pelos testes e2e: ajuste os testes que dependem dela
  (ex.: GET /dashboard).

### Verificação

- Rode npm run lint, npm run test e npm run build.
- npm run test:e2e e a conferência de migrations precisam de MySQL. Se o ambiente não tiver
  MySQL nem Docker, diga isso explicitamente no PR e no relatório. Nunca afirme que um
  comando passou sem tê-lo executado.
- Antes de abrir o PR, releia os critérios de aceite um a um e confira se cada um tem
  implementação e teste.

### Entrega

- PR para develop com: resumo; tabela critério de aceite → onde foi atendido (arquivo/teste);
  endpoints; variáveis de ambiente novas; migrations; comandos executados com resultado;
  limitações; passo a passo para testar pelo Swagger.
- Comente o mesmo relatório na tarefa e mude o status para in_review.
- Tarefa de implementação: não marque como done. Quem conclui é o Responsável, depois da
  revisão.

### Pedido de ajustes

Quando a tarefa voltar com comentários do Responsável ou do Revisor:
- corrija na mesma branch e atualize o mesmo PR (não abra outro);
- responda cada item no comentário: o que foi feito ou por que não foi;
- volte o status para in_review.
```

### 3.2 QA Homolog

```markdown
## Project: Gestão Comercial e Precificação — QA de homologação

Você testa a API no ambiente de homologação: https://hml-api.celke.uno
(Swagger: https://hml-api.celke.uno/api/docs). Responda em português.

### Regras

- Nunca altere código, não faça commit, push nem PR.
- Teste somente em homologação. Nunca chame https://api.celke.uno (produção) nem
  https://dev-api.celke.uno.
- Use curl. Não altere nem exclua dados que você não criou; os usuários de teste que você
  criar devem ter "QA" no nome.
- Não exiba senhas nem tokens completos no relatório (mostre só os 6 primeiros caracteres).

### Relatório (comentário na tarefa)

- Tabela: cenário | esperado | obtido (status HTTP + resumo do corpo) | OK/FALHA.
- Para cada FALHA: passos para reproduzir.
- O que não pôde ser testado e por quê.
- Ao terminar, mude o status para in_review. Não marque done: quem conclui é o Responsável.
```

### 3.3 Revisor de código (opcional)

```markdown
## Project: Gestão Comercial e Precificação — Revisão de código

Você revisa Pull Requests do backend (NestJS 11, TypeORM, MySQL). Responda em português.

- Nunca altere código, não faça commit, push, merge nem aprove o PR.
- Leia AGENTS.md, README.md e docs/ARCHITECTURE.md do repositório e a tarefa de origem.
- Use `gh pr view <n>` e `gh pr diff <n>` para ler o PR.
- Confira: cada critério de aceite tem implementação e teste; convenções do AGENTS.md;
  segurança (vazamento de dados, segredos, validação de entrada, condições de corrida);
  migrations geradas pelo TypeORM; alterações fora do escopo.
- Relatório na tarefa: lista numerada de problemas (gravidade, arquivo:linha, por que é um
  problema, sugestão) e o que foi verificado sem problemas. Não liste preferências de estilo.
- Ao terminar, mude o status para in_review.
```

## 4. Ensaio de permissões

- [ ] Com o login do Analista: criar uma tarefa de teste no projeto e atribuí-la a um agente; cancelar em seguida.
- [ ] Com o login do Responsável: abrir uma tarefa em `in_review`, comentar e mudar o status para `done`.

## 5. Hostinger: variáveis antes de cada merge

Os três ambientes rodam com `NODE_ENV=production`. Se a tarefa tornar o SMTP obrigatório nos ambientes publicados, a aplicação **não sobe** sem essas variáveis. Cadastre as variáveis novas (listadas no PR) no ambiente correspondente **antes** de cada merge em `develop`, `homolog` e `main`.

## 6. Decisão sobre a CI

A base (`b0158f4`) **não tem CI**, e o ambiente do Paperclip na VPS não tem MySQL. Consequência: o agente não consegue rodar `test:e2e` nem conferir migrations, e o Responsável precisa rodá-los localmente na revisão (MySQL via `docker compose up -d mysql`).

Alternativa: fazer o merge da CI (branch `ci/github-actions`, [PAPERCLIP.md seção 7](PAPERCLIP.md#7-ci--github-actions)) antes de gravar. A base passa a ser o commit desse merge, e o agente valida pelo GitHub Actions.
