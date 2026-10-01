# varal-infra

Infraestrutura do Varal: Docker Compose, NGINX e ambiente de desenvolvimento.

Specs e decisões do produto: [varal-docs](https://github.com/mchlima/varal-docs).

## Postgres de desenvolvimento

Um único Postgres 17 local atende todos os worktrees da API (spec 01, seção 4.1).

```sh
docker compose -f dev/compose.yml up -d     # sobe (porta 5432, só em 127.0.0.1)
docker compose -f dev/compose.yml ps        # situação
```

- Usuário e senha `varal` / `varal` (só desenvolvimento). URL base: `postgresql://varal:varal@localhost:5432/postgres`.
- Cada worktree do `varal-web-api` cria os próprios bancos (`varal_<slug>` e `varal_<slug>_test`) com o `scripts/worktree.sh`.
- Nunca rode `docker compose -f dev/compose.yml down -v` nem apague o volume `varal-dev-db-data`: isso apaga os bancos de todos.

## E-mail de desenvolvimento (Mailpit)

O mesmo Compose sobe o Mailpit, que recebe os e-mails enviados pela API em desenvolvimento e não entrega nada a ninguém.

- SMTP: `localhost:1025`, sem autenticação e sem TLS. Use na API: `SMTP_HOST=localhost`, `SMTP_PORT=1025`.
- Caixa de entrada: http://localhost:8025

## NGINX

Hosts do Varal no NGINX compartilhado do VPS: ver [`nginx/README.md`](nginx/README.md).
