# NGINX do Varal

Os arquivos de `conf.d/` são os hosts do Varal no NGINX compartilhado do VPS (`/opt/nginx`, fora deste repositório; spec 01, seção 4). O Cloudflare fica na frente, com proxy ligado e SSL Full (strict).

| Arquivo | Host | Serve |
| --- | --- | --- |
| `varal-varal.kratinho.com.br.conf` | `varal.kratinho.com.br` | front do painel (`varal-panel-web`) |
| `varal-admin-varal.kratinho.com.br.conf` | `admin-varal.kratinho.com.br` | front do admin (`varal-admin-web`) |
| `varal-api-web-varal.kratinho.com.br.conf` | `api-web-varal.kratinho.com.br` | backend: API REST (`/api/v1`) e WebSocket (`/ws`) |

## O que os arquivos esperam do VPS

- Certificado de origem do Cloudflare (`*.kratinho.com.br`, válido até 2041) em `/opt/nginx/certs/kratinho.com.br/origin.pem` e `origin.key`. Eles nunca entram no repositório.
- `/opt/nginx/snippets/cloudflare-realip.conf`, compartilhado, com as faixas de IP do Cloudflare.
- Builds estáticos em `/opt/nginx/html/varal-<app>/releases/<versão>/`, com o link `current` apontando para a versão no ar (trocado pelo `deploy-app`, ver `prod/README.md`).
- API no container `varal-api`, porta 3000, na rede `proxy`. Enquanto ela não existe, as rotas da API respondem 502.

## Aplicar no VPS

Só com pedido explícito do usuário.

```sh
( set -a; . ~/.config/varal/credentials/vps-sv-general-00.env; set +a
  SSHPASS="$VPS_PASSWORD" sshpass -e scp nginx/conf.d/varal-*.conf "$VPS_USER@$VPS_HOST:/opt/nginx/conf.d/"
  SSHPASS="$VPS_PASSWORD" sshpass -e ssh "$VPS_USER@$VPS_HOST" 'docker exec nginx nginx -t && docker exec nginx nginx -s reload' )
```

Se o `nginx -t` falhar, não recarregue: corrija e envie de novo. Nunca reinicie nem recrie o container para aplicar configuração.
