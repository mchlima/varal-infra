# Produção

Como o Varal roda no VPS (spec 01, seção 4) e como uma versão chega lá (plano, seção 2.4).

| No VPS | O que é |
| --- | --- |
| `/opt/varal/compose.yml` | Compose da API (`prod/compose.yml`) |
| `/opt/varal/.env` | Variáveis da API (modelo em `prod/.env.example`; dono `root`, grupo `deploy`, `640`) |
| `/opt/varal/bin/` | Scripts de `prod/bin/` (dono `root`, `755`) |
| `/opt/varal/state/api-version` | Versão da API no ar |
| `/opt/nginx/html/varal-<app>/releases/<versão>/` | Builds dos apps; `current` aponta para a versão no ar |
| Usuário `deploy` | Grupo `docker`; a chave dele só roda o `deploy-gate` |

PostgreSQL: banco `varal` e usuário `varal` no container compartilhado `postgres` (RN-01.16), com limite de 12 conexões.

## Deploy

O merge do PR de versão do release-please cria a tag `vX.Y.Z`, e o mesmo workflow (`release-please.yml` de cada repositório de código) publica a versão pelo SSH do usuário `deploy`:

```sh
docker save varal-web-api:vX.Y.Z | gzip | ssh deploy@<vps> api vX.Y.Z      # varal-web-api
tar -czf - -C .output/public . | ssh deploy@<vps> app panel vX.Y.Z           # varal-panel-web (admin: app admin)
ssh deploy@<vps> status                                                       # versões no ar
```

- **API** (`deploy-api`): carrega a imagem, roda `prisma migrate deploy` num container temporário e só então troca o `varal-api`. Se a versão nova não ficar saudável em 90 s, volta para a anterior. Guarda as 3 imagens mais recentes.
- **Apps** (`deploy-app`): extrai em `releases/<versão>` e troca o link `current` de uma vez; guarda as 5 versões mais recentes.
- O `deploy-gate` é o forced command da chave do `deploy` no `authorized_keys` (`restrict,command="/opt/varal/bin/deploy-gate"`): aceita só `api`, `app` e `status`.
- O mesmo comando funciona à mão, com a chave de deploy das credenciais locais. Para voltar um app de versão sem novo build, no VPS: `ln -sfn releases/<versão> current.new && mv -T current.new current`.

Os segredos do GitHub em cada repositório de código: `DEPLOY_SSH_KEY` (chave privada do `deploy`), `DEPLOY_HOST` e `DEPLOY_KNOWN_HOSTS`.
