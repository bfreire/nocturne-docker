# nocturne-docker

Stack Docker pronta a puxar pelo **Portainer** para correr o [Nocturne](https://github.com/nightscout/nocturne) — a reescrita em .NET 10 da API do Nightscout — usando as imagens oficiais publicadas no GHCR pela CI do projecto upstream. Nenhum código do Nocturne é alterado: só wrap.

> Imagens consumidas: `ghcr.io/nightscout/nocturne/nocturne-api`. Base de dados: PostgreSQL 17.6 com os três roles oficiais (`nocturne_migrator`, `nocturne_app`, `nocturne_web`) provisionados pelo script `pg-init/00-init.sh` (cópia *byte-a-byte* de [`docs/postgres/container-init/00-init.sh`](https://github.com/nightscout/nocturne/blob/main/docs/postgres/container-init/00-init.sh)).

## Conteúdo do repo

| Ficheiro | O que faz |
| --- | --- |
| `docker-compose.yml` | Stack: `postgres` + `nocturne-api` + `watchtower` (com label scope). |
| `.env.example` | Template das variáveis. Copia para `.env` e preenche. |
| `postgres/Dockerfile` + `postgres/00-init.sh` | Imagem custom do Postgres com o script dos três roles já dentro (`/docker-entrypoint-initdb.d/`). Construída em build-time pelo Portainer — evita o bind-mount que dá `mkdir /data: read-only file system`. |
| `.gitignore` | Garante que `.env` nunca vai para o repo. |

## Rede e acesso externo

A stack NÃO publica nenhuma porta no host por defeito — o acesso externo passa por um reverse proxy (Caddy) que se liga à mesma rede Docker. Ver secção "Configuração da Caddy" abaixo.

Se quiseres fazer debug local sem Caddy, descomenta o bloco `ports:` no `docker-compose.yml` (já lá deixei o snippet) e acede em `http://127.0.0.1:8731`.

> Kestrel está em HTTP, não HTTPS. A imagem oficial do Nocturne não traz certificado TLS, e `HTTPS_PORTS` rebenta logo no arranque com *"No server certificate was specified"*. A TLS é terminada pela Caddy.

## Como subir no Portainer (Stack from Git)

1. **Portainer → Stacks → Add stack → Repository**.
2. **Repository URL**: `https://github.com/bfreire/nocturne-docker` (depois de fazeres o push abaixo).
3. **Compose path**: `docker-compose.yml`.
4. **Environment variables**: copia as chaves do `.env.example` e preenche os valores. **Não** uses o ficheiro `.env` por upload — Portainer prefere envs colados no formulário (assim ficam no Portainer e não no repo).
5. **Deploy the stack**.

Notas:

- O `postgres/00-init.sh` é incluído numa imagem custom construída em build-time (`build: ./postgres`). Isto contorna o bug típico do Portainer com Stacks from Git, em que bind mounts relativos resolvem para `/data/compose/<id>/...` dentro do container do Portainer mas não existem no host onde corre o daemon do Docker (erro `mkdir /data: read-only file system`).
- Na primeira arranque, o Postgres só executa o init script se o volume `nocturne-postgres-data` estiver vazio. Se mudares as passwords depois, tens de fazer `docker compose down -v` (apaga dados!) ou aplicá-las manualmente via `ALTER ROLE`.

## Gerar segredos rapidamente

```bash
openssl rand -base64 32   # repete para cada *_PASSWORD e o INSTANCE_KEY
```

Mínimo de quatro execuções:

| Variável | Para que serve |
| --- | --- |
| `POSTGRES_PASSWORD` | Superuser de bootstrap (só usado uma vez na criação do cluster). |
| `NOCTURNE_MIGRATOR_PASSWORD` | Role que aplica migrations (DDL). |
| `NOCTURNE_APP_PASSWORD` | Role usado pela API em runtime (com RLS). |
| `NOCTURNE_WEB_PASSWORD` | Role do framework de bots (tabelas `chat_state_*`). |
| `INSTANCE_KEY` | Assina JWTs e cookies do Nocturne. |

## Configuração da Caddy

A stack do Nocturne cria uma rede Docker chamada **`nocturne-net`**. Para a Caddy chegar ao API basta entrar nessa rede.

### 1. Liga a Caddy à rede `nocturne-net`

No `docker-compose.yml` da tua stack da Caddy, adiciona:

```yaml
services:
  caddy:
    # ... a tua config existente ...
    networks:
      - default          # mantém a rede original da Caddy
      - nocturne-net     # liga à rede do Nocturne

networks:
  nocturne-net:
    external: true       # rede gerida pela stack do Nocturne
```

Faz redeploy da stack da Caddy. A partir daqui a Caddy resolve `nocturne-api` via DNS interno do Docker.

> **Ordem importa**: a stack do Nocturne tem de estar a correr ANTES de a stack da Caddy ser redeployed, senão a rede `nocturne-net` ainda não existe e o redeploy falha com `network nocturne-net declared as external, but could not be found`.

### 2. Adiciona o bloco ao Caddyfile

Vê o ficheiro [`Caddyfile.example`](./Caddyfile.example) no repo. Copia o bloco, substitui `nocturne.example.com` pelo teu subdomínio real, e cola no Caddyfile da tua stack.

Exemplo mínimo:

```caddy
nocturne.example.com {
    reverse_proxy nocturne-api:8080
}
```

A Caddy trata automaticamente do certificado Let's Encrypt — desde que o subdomínio resolva para o IP público da máquina e as portas 80/443 estejam acessíveis.

### 3. Recarrega a Caddy

```bash
docker exec <caddy-container> caddy reload --config /etc/caddy/Caddyfile
```

ou simplesmente reinicia o container.

### 4. Testa

```bash
curl -I https://nocturne.example.com/api/v1/status
```

Deves ver `HTTP/2 200` (ou 401 se o endpoint exigir auth — também conta como sinal de que o reverse proxy funciona).

## Updates automáticos

Watchtower está incluído, mas com `WATCHTOWER_LABEL_ENABLE=true` — só toca em containers com o label `com.centurylinklabs.watchtower.enable=true`. Por defeito só o `nocturne-api` tem o label, logo Postgres e o próprio Watchtower ficam estáveis. Polling: 24h.

Para fazer update manual:

```bash
docker compose pull && docker compose up -d
```

## Voltar atrás

```bash
docker compose down       # mantém os dados
docker compose down -v    # apaga o volume da BD (PERDE TUDO)
```

## Disclaimer

Mesmas regras do upstream: este software é fornecido *as-is* para uso pessoal. Não usa este wrapper para tomar decisões médicas — confirma sempre as leituras de glicemia com dispositivos médicos aprovados.
