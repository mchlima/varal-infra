# AGENTS.md

Instruções para agentes de código (Claude Code, Codex, Cursor e similares) que trabalham no `varal-infra`.

> Arquivo gerado a partir do [varal-docs](https://github.com/mchlima/varal-docs/tree/main/docs/agents) (`docs/agents/varal-infra.md` + `docs/agents/regras-comuns.md`). Não edite aqui: mude no varal-docs e regenere com `scripts/build-agents.sh varal-infra`.

## Este repositório: varal-infra

Infraestrutura do Varal: Docker Compose de produção no VPS, configuração do Varal no NGINX compartilhado (HTTPS de origem, proxy para a API e servidor dos builds dos apps), atrás do Cloudflare (DNS, proxy e HTTPS público), backup do PostgreSQL e o Postgres de desenvolvimento compartilhado pelos worktrees da API. Spec principal: 01, seções 4 e 4.1.

### Comandos

```sh
docker compose -f dev/compose.yml up -d    # Postgres de desenvolvimento (varal-dev-db, 127.0.0.1:5432, varal/varal)
docker compose -f dev/compose.yml ps
```

Configuração do NGINX e como aplicá-la no VPS: `nginx/README.md`. Deploy: a definir (plano de desenvolvimento, seção 6).

### Regras deste repositório

- **Produção:** nenhuma mudança é aplicada no VPS sem pedido explícito do usuário. Este repositório descreve a infraestrutura; aplicar é uma ação à parte.
- **Tudo em Docker:** no VPS, tudo roda em container; nada é instalado direto no host. O PostgreSQL e o proxy NGINX do VPS são **compartilhados com outros projetos** e ficam fora deste repositório (o PostgreSQL em `/opt/postgres` e o NGINX em `/opt/nginx`, ambos descritos na spec 01, seção 4). O Compose de produção do Varal não define `nginx` nem `postgres`: a API entra nas redes Docker externas `postgres` e `proxy`, usa banco e usuário `varal`, e o Varal entra no proxy com um `server` por host. Nunca altere, reinicie ou recrie os serviços compartilhados nem mexa na configuração de outros projetos sem pedido explícito; para aplicar mudanças no NGINX, valide (`nginx -t`) e recarregue (`nginx -s reload`), nunca reinicie.
- **Cloudflare:** os hosts do Varal ficam com proxy ligado e SSL Full (strict). O NGINX usa o Cloudflare Origin Certificate curinga de `/opt/nginx/certs/kratinho.com.br/`, nunca o certbot, e restaura o IP real pelo `CF-Connecting-IP` (spec 01, RN-01.18 e RN-01.19). Certificados e chaves nunca entram no repositório.
- **Hosts:** `varal.kratinho.com.br` (front do painel), `admin-varal.kratinho.com.br` (front do admin) e `api-web-varal.kratinho.com.br` (API). Os arquivos de cada host ficam em `nginx/conf.d/` e são copiados para `/opt/nginx/conf.d/` (passo a passo em `nginx/README.md`).
- **Segredos** nunca entram no repositório: só `.env.example` com nomes e descrições.
- **Versões, não código:** o Compose de produção referencia imagens ou builds dos outros repositórios por tag; não copie código deles para cá.
- **Postgres de desenvolvimento** (`dev/compose.yml`, projeto `varal-dev-db`): é compartilhado por todos os worktrees de todos os agentes. Nunca rode `docker compose down -v`, apague o volume nem recrie o container sem pedido explícito, porque isso destrói os bancos de todos.
- **Backup:** fora do MVP por enquanto (spec 01, seção 4). Quando entrar, vem com o procedimento de restauração testado e documentado.

Escopos de commit adicionais: `nginx`, `compose`, `backup`, `dev`, `deploy`.

## O projeto Varal

Varal é um SaaS de assinatura mensal para barracas de feirinha (espetos, pastéis, tapiocas). Cada colaborador usa o próprio celular como estação de trabalho: o balcão registra o pedido e ele cai direto na tela da cozinha, em tempo real. O sistema cobre comandas, pagamentos registrados, caixa, fiado e relatórios, além de um admin para a equipe do Varal. Piloto: um vendedor de espetos de churrasco.

### Repositórios

| Repositório | Conteúdo |
| --- | --- |
| [`varal-docs`](https://github.com/mchlima/varal-docs) | Specs, glossário, decisões e estas regras comuns |
| [`varal-web-api`](https://github.com/mchlima/varal-web-api) | API NestJS (REST `/api/v1` e WebSocket `/ws`), banco e migrations, `openapi.json` |
| [`varal-panel-web`](https://github.com/mchlima/varal-panel-web) | App Nuxt dos clientes (PWA): balcão, estações, caixa, painel do dono |
| [`varal-admin-web`](https://github.com/mchlima/varal-admin-web) | App Nuxt do admin da plataforma |
| [`varal-infra`](https://github.com/mchlima/varal-infra) | Docker Compose de produção, NGINX, Postgres de desenvolvimento |

Localmente, os repositórios ficam lado a lado numa pasta comum (`varal/varal-docs`, `varal/varal-web-api`…). Use os caminhos relativos `../varal-docs` etc. para ler os vizinhos; nunca edite um repositório vizinho a partir de outro.

## Specs: a fonte da verdade

- Todas as regras de produto estão nas specs do `varal-docs` ([`docs/specs/`](https://github.com/mchlima/varal-docs/tree/main/docs/specs), localmente `../varal-docs/docs/specs/`). Leia o README das specs (convenções e glossário) e a spec do módulo antes de implementar qualquer coisa.
- Regras de negócio têm id `RN-XX.YY` e critérios de aceite `CA-XX.YY`. Cite os ids nos testes, nos comentários que explicam uma regra e nas mensagens de commit.
- Cada critério de aceite implementado tem pelo menos um teste automatizado.
- Itens marcados **(proposta)** ainda não foram confirmados pelo dono do projeto. Não trate como decisão definitiva; se a implementação depender de um deles, pergunte.
- Se o comportamento precisar divergir da spec, abra também um PR no `varal-docs` atualizando a spec, e cite um PR no outro. Nunca deixe código e spec contando histórias diferentes.
- Não use frameworks de planejamento externos (como GSD). Planos e specs são arquivos markdown no `varal-docs`.

## Contratos entre repositórios

- A API é a fonte única dos contratos. O `varal-web-api` mantém o `openapi.json` commitado, com rotas, schemas, enums de estado e payloads dos eventos em tempo real (spec 01, seção 3.1).
- Os apps geram tipos e cliente HTTP a partir do `openapi.json` (`pnpm gen:api`). Tipos gerados nunca são editados à mão, e nenhum enum ou formato de evento é redefinido manualmente num app.
- Mudanças na API são compatíveis com versões anteriores sempre que possível (só adicionar). Mudança incompatível exige PRs coordenados e o rodapé `BREAKING CHANGE`.

### Mudanças que atravessam repositórios

- Use **o mesmo nome de branch** em todos os repositórios afetados (ex.: `feat/reabrir-comanda` no `varal-docs`, no `varal-web-api` e no `varal-panel-web`).
- Cada PR lista, na descrição, os PRs relacionados nos outros repositórios.
- Ordem de merge: spec (`varal-docs`) → API (`varal-web-api`) → apps (`varal-panel-web`, `varal-admin-web`) → infra (`varal-infra`), quando houver.
- Um agente trabalha em um repositório por vez; se a tarefa exigir mudar outro repositório, abra o worktree e a branch lá também, seguindo as mesmas regras.

## Convenções de código

- **Idioma:**
  - **Inglês** em toda a codebase: variáveis, funções, classes, componentes, arquivos, tabelas e colunas, rotas da API (`/api/v1/tabs`), eventos, enums e chaves de permissão. Use os nomes do glossário das specs (ex.: comanda = `Tab`, turno = `Shift`, fiado = `on_credit`).
  - **Português do Brasil** em tudo que o usuário vê: textos da interface, mensagens de erro exibidas e **rotas do front** (`/balcao`, `/painel/cardapio`). Specs e mensagens de commit também em português.
  - No Nuxt, os arquivos em `pages/` seguem o nome da rota em português (`pages/balcao.vue`), por ser o roteamento por arquivo. É a única exceção ao inglês; componentes, composables e stores continuam em inglês.
- **TypeScript estrito.** Sem `any` sem justificativa.
- **Dinheiro:** sempre inteiro em centavos, campos com sufixo `_cents` / `Cents`. Nunca `float` ou `decimal` em JavaScript.
- **Datas:** `timestamptz` em UTC no banco; exibição em `America/Sao_Paulo`.
- **IDs:** UUID v7.
- **Nada operacional é apagado:** comandas, pedidos, itens, pagamentos e movimentos de caixa são cancelados ou estornados, nunca removidos. Cadastros são desativados.
- **Segredos** só em variáveis de ambiente. Nunca commitar `.env`, credenciais de SMTP ou chaves.

## Credenciais locais

Credenciais de infraestrutura (VPS, PostgreSQL de produção e outras) ficam **só na máquina de desenvolvimento**, fora de qualquer repositório:

- Pasta: `~/.config/varal/credentials/` (permissão `700`, arquivos `600`).
- Comece pelo `README.md` da pasta: ele lista cada arquivo `.env`, o que contém e o nome das variáveis.
- Carregue as variáveis num subshell, sem imprimir os valores: `( set -a; . ~/.config/varal/credentials/postgres.env; set +a; <comando> )`.
- **Nunca** copie valores dessa pasta para repositórios, commits, PRs, issues, logs, memória do agente ou mensagens, e nunca os imprima (`cat`, `echo`, `env`).
- Ações no VPS ou no banco de produção só com pedido explícito do usuário.
- Se faltar uma credencial, peça ao usuário; não invente nem procure em outros lugares.

## Vários agentes ao mesmo tempo

Cada repositório é trabalhado por vários agentes em paralelo. Para que um não atrapalhe o outro:

### Uma branch por assunto, um agente por branch

- Cada agente desenvolve na **branch do assunto em que está trabalhando** (ex.: `feat/fechamento-de-caixa`), nunca na `main` e nunca na branch de outro agente.
- Uma branch trata de **um único assunto**. Se aparecer algo de outro assunto no meio do trabalho, anote para o usuário ou abra outra branch; não misture no mesmo PR.
- Assuntos diferentes = branches diferentes, mesmo que o mesmo agente trabalhe em ambos.

### Cada agente no seu worktree

- **O checkout principal (a raiz do repositório) fica sempre na `main`, limpo.** Ninguém edita arquivos nem troca de branch nele; ele só serve de base.
- Cada tarefa roda num **git worktree próprio**, com a sua branch, **fora do repositório**: na pasta comum `varal/.worktrees/<repositório>/<tipo>-<descricao>` (ao lado dos repositórios, fora de qualquer git). Rodando da raiz do repositório:
  ```
  git fetch origin && git worktree add ../.worktrees/<repositório>/<tipo>-<descricao> -b <tipo>/<descricao> origin/main
  ```
  Nos repositórios de código, use `scripts/worktree.sh new <tipo>/<descricao>`, que cria o worktree nesse lugar e também prepara portas e banco (spec 01, seção 4.1).
- **Nunca crie worktree dentro do repositório** (nem em `.worktrees/`, nem em `.claude/worktrees/`, que é onde a ferramenta `EnterWorktree` do Claude Code cria). Ferramentas que sobem pelas pastas procurando configuração (Nuxt, Vite, TypeScript) encontrariam a do checkout principal, e o build e os testes quebram.
- Trabalhe, rode comandos e faça commits **somente dentro do seu worktree**. Nunca edite arquivos de outro worktree nem da raiz.
- Ao terminar (PR aceito ou tarefa abandonada), remova o worktree (`scripts/worktree.sh remove <nome>` ou `git worktree remove ../.worktrees/<repositório>/<nome>`).

### Antes de começar

- Rode `git worktree list` para ver as branches em andamento. Não pegue uma tarefa ou módulo que já tenha branch aberta; se precisar mexer no mesmo módulo, combine com o usuário.
- Prefira tarefas pequenas e de um módulo só. PRs pequenos reduzem conflito.

### Comandos proibidos fora do seu worktree

Estes comandos afetam o trabalho de outros agentes e só podem ser usados dentro do seu próprio worktree, nunca na raiz:
`git switch`/`git checkout` de branch, `git stash`, `git reset --hard`, `git clean`, `git rebase`, `git restore` em massa, `rm -rf` em pastas do projeto.

Também é proibido: apagar ou mover worktrees de outros, apagar branches que não são suas, `git push --force` em branch que não é sua, e alterar a configuração global do git.

### Ambiente isolado por worktree

Cada worktree roda o próprio ambiente sem disputar portas nem banco com os outros (spec 01, seção 4.1): portas definidas pelo `PORT_OFFSET` no `.env.local` do worktree e, na API, banco próprio no Postgres de desenvolvimento compartilhado. Nunca rode migrations, seeds ou `docker compose down -v` apontando para o banco ou o projeto de outro worktree.

## Commits e branches

### Fluxo de branches

- **Nada vai direto para a `main`.** Nem commit, nem push, nem merge local. Toda mudança nasce numa branch de trabalho e entra na `main` por pull request.
- **Nome da branch:** `<tipo>/<descricao-curta>`, com o mesmo `tipo` do Conventional Commits e descrição em minúsculas com hífens. Ex.: `feat/reabrir-comanda`, `fix/troco-em-dinheiro`, `docs/spec-fiado`.
- O PR vai da branch de trabalho para a `main`, com título no formato Conventional Commits e descrição com o que mudou, as regras e critérios de aceite cobertos (`RN-XX.YY`, `CA-XX.YY`), como testar e os PRs relacionados em outros repositórios.
- Não faça commit, push nem abra PR sem pedido explícito.

### Bloqueio da main

Como o plano gratuito do GitHub não protege branches em repositórios privados, o bloqueio é feito localmente, em duas camadas presentes em todos os repositórios:

- **Git hooks** em `.githooks/`: `pre-commit` recusa commit na `main` e `pre-push` recusa push para a `main`. Ative uma vez por clone com `git config core.hooksPath .githooks` (vale para todos os worktrees daquele clone). O `scripts/worktree.sh` dos repositórios de código confere isso.
- **Hook do Claude Code** em `.claude/settings.json` (`.claude/hooks/guard-main.py`): antes de executar, recusa `git commit`, `merge`, `rebase`, `cherry-pick`, `revert` e `am` na `main`, push para a `main` e qualquer `--no-verify`.
- **Nunca** use `--no-verify`, nunca desative ou altere os hooks para contornar o bloqueio, e nunca mude `core.hooksPath`. Se um hook bloquear algo legítimo, pare e fale com o usuário.

### Mensagens de commit

Todo commit segue o [Conventional Commits 1.0.0](https://www.conventionalcommits.org/pt-br/v1.0.0/):

```
<tipo>(<escopo opcional>): <descrição>

<corpo opcional>

<rodapés opcionais>
```

- **Tipos:** `feat` (funcionalidade), `fix` (correção), `docs` (documentação e specs), `refactor`, `test`, `perf`, `style` (formatação, sem mudar comportamento), `build` (dependências, empacotamento), `ci`, `chore` (manutenção).
- **Escopo:** o módulo afetado, em inglês e minúsculas: `auth`, `tenancy`, `audit`, `email`, `realtime`, `rbac`, `organizations`, `announcements`, `metrics`, `impersonation`, `units`, `workflow`, `menu`, `staff`, `shifts`, `tabs`, `orders`, `cash`, `credit`, `reports`, `ui`. Cada repositório pode acrescentar os seus (ver a seção específica dele).
- **Descrição:** em português, no imperativo, minúscula no início, sem ponto final, até cerca de 72 caracteres. Cite a regra quando houver: `feat(tabs): permite reabrir comanda em fechamento (RN-04.12)`.
- **Mudança incompatível:** `!` depois do tipo/escopo e rodapé `BREAKING CHANGE: <explicação>`.
- Um commit por mudança coerente.
