# Guia para agentes de IA e desenvolvedores

Convenções deste repositório. Siga-as ao implementar qualquer tarefa.

## Antes de começar

- Leia `README.md`, `docs/ARCHITECTURE.md` e `docs/ROADMAP.md`.
- Configuração do Paperclip, agentes e tarefas em andamento: `docs/PAPERCLIP.md`.
- Ambiente: `npm install`, `.env` a partir do `.env.example`, `docker compose up -d mysql`, `npm run db:setup`.

## Branches

- `main` = produção, `homolog` = testes humanos, `develop` = integração.
- Trabalhe sempre na branch da tarefa, criada a partir de `develop`, e entregue por Pull Request para `develop`. No Paperclip, a branch já vem criada (`<ID>-<slug>`, ex.: `CEL-3-...`): use-a e não crie outra. Fora dele, crie uma (ex.: `feature/recuperacao-de-senha`).
- **Nunca** faça commit ou push diretamente em `develop`, `homolog` ou `main`. As promoções `develop → homolog → main` são feitas por Pull Request, após os testes humanos.

## Definição de pronto

Toda tarefa só está concluída quando estes comandos passam:

```bash
npm run lint
npm run test
npm run test:e2e
npm run build
```

E, se houver alteração de entidades, quando `npm run typeorm -- migration:generate src/database/migrations/Check --dryrun` responder que não há mudanças pendentes.

### CI

O GitHub Actions ([.github/workflows/ci.yml](.github/workflows/ci.yml)) roda esses mesmos comandos, com MySQL, em todo Pull Request e a cada push em `develop`, `homolog` e `main`. A conferência de migrations usa `migration:generate --check`, que falha se houver pendência.

- Rode localmente tudo o que o ambiente permitir antes de abrir o PR.
- Se algum comando não puder rodar no seu ambiente (ex.: `test:e2e` sem MySQL), diga isso no PR e use a CI como verificação: acompanhe com `gh pr checks <número> --watch`.
- CI vermelha: leia o log da falha (`gh run view <id> --log-failed`), corrija na mesma branch e envie de novo. Um PR só está pronto para revisão com a CI verde.

## Padrões de código

- **Idioma:** código (nomes de classes, métodos, variáveis, rotas) em inglês; mensagens ao usuário, comentários e documentação em português.
- **Controllers** só lidam com HTTP (rota, params, status, Swagger). Toda regra de negócio fica no **service**.
- **Services** lançam exceções do Nest (`NotFoundException`, `ConflictException`, `BadRequestException`...). O `AllExceptionsFilter` padroniza a resposta.
- **DTOs** validam tudo com `class-validator` e documentam com `@ApiProperty`. Use `PartialType`/`PickType` de `@nestjs/swagger` (não de `@nestjs/mapped-types`).
- **Entidades** estendem `AppBaseEntity`; índices únicos com nome explícito `UQ_<tabela>_<colunas>`; colunas em `snake_case` via `name:` quando tiverem mais de uma palavra.
- **Um módulo não acessa o repositório de outro** — use o service exportado.
- **Rotas são privadas por padrão.** Use `@Public()` apenas quando necessário e `@Roles(Role.Admin)` para restringir.
- **Senhas:** sempre via `PasswordService` e `UsersService.updatePassword()`; validação com `@IsStrongPassword()`.
- **Swagger:** toda rota tem `@ApiOperation`, resposta de sucesso tipada e `@ApiErrorResponses(...)`.
- **Configuração:** nova variável de ambiente → declarar em `src/config/env.validation.ts` e no `.env.example`. Nunca coloque segredos no código.

## Checklist para um novo módulo

1. `src/<modulo>/` com `entities/`, `dto/`, `<modulo>.service.ts`, `<modulo>.controller.ts`, `<modulo>.module.ts`.
2. Registrar o módulo em `src/app.module.ts`.
3. Gerar migration: `npm run migration:generate -- src/database/migrations/<Nome>`; revisar o SQL gerado.
4. Testes unitários do service (`*.spec.ts`, use `test/utils/repository.mock.ts`) e cenários e2e em `test/`.
5. Se fizer sentido para demonstração, adicionar dados em `src/database/seeds/seed-data.ts`.
6. Atualizar o README (endpoints) e, se for o caso, o `docs/`.

## Funcionalidades pendentes

A **recuperação de senha** ainda não existe e é uma tarefa planejada — veja `docs/ROADMAP.md`.
