# nocturne-docker

Stack Docker pronta a puxar pelo **Portainer** para correr o [Nocturne](https://github.com/nightscout/nocturne) — a reescrita em .NET 10 da API do Nightscout — **compilada a partir do código-fonte** do upstream. Nenhum código do Nocturne é alterado: só wrap + Dockerfiles de build.

> **Build strategy:** `nocturne-api` é compilado por `api/Dockerfile` (neste repo) via `dotnet publish` — o upstream não tem Dockerfile para a API. `nocturne-web` usa o `Dockerfile.web` do upstream directamente como contexto Git remoto. Base de dados: PostgreSQL 17.6 com os três roles oficiais (`nocturne_migrator`, `nocturne_app`, `nocturne_web`) provisionados pelo script `postgres/00-init.sh` (cópia *byte-a-byte* de [`docs/postgres/container-init/00-init.sh`](https://github.com/nightscout/nocturne/blob/main/docs/postgres/container-init/00-init.sh)).

## Conteúdo do repo

| Ficheiro | O que faz |
| --- | --- |
| `docker-compose.yml` | Stack: `postgres` + `nocturne-api` + `nocturne-web` + `watchtower`. |
| `.env.example` | Template das variáveis. Copia para `.env` e preenche. |
| `api/Dockerfile` | Build da API .NET 10 a partir do upstream (git clone + dotnet publish). O upstream não tem Dockerfile para a API — usa `dotnet publish --container` em CI. |
| `postgres/Dockerfile` + `postgres/00-init.sh` | Imagem custom do Postgres com o script dos três roles já dentro (`/docker-entrypoint-initdb.d/`). Construída em build-time pelo Portainer — evita o bind-mount que dá `mkdir /data: read-only file system`. |
| `Caddyfile.example` | Bloco pronto a colar no Caddyfile (loopback pattern). |
| `.gitignore` | Garante que `.env` nunca vai para o repo. |

## Arquitectura

```
                      ┌──────────────┐
   browser ──HTTPS──▶ │    Caddy     │   (host network, termina TLS)
                      └──────┬───────┘
                             │ http://127.0.0.1:8731
                             ▼
                      ┌──────────────────────┐
                      │  nocturne-web (1612) │   SvelteKit UI + bot framework
                      └──────┬───────────────┘
                             │ http://nocturne-api:8080  (Docker DNS)
                             ▼
                      ┌──────────────────────┐
                      │  nocturne-api (8080) │   .NET REST API (Nightscout-compat)
                      └──────┬───────────────┘
                             │ Host=nocturne-postgres-server (Docker DNS)
                             ▼
                      ┌─────────────────────────────────┐
                      │ nocturne-postgres-server (5432) │   3 roles: migrator/app/web
                      └─────────────────────────────────┘
```

A API **não** está exposta no host por defeito — o browser nunca chama directamente. Quem chama é o web (server-side via `NOCTURNE_API_URL`). Se quiseres expor a API diretamente para uploaders Nightscout (AndroidAPS, xDrip+), descomenta o bloco `ports:` do `nocturne-api` no `docker-compose.yml` e o segundo bloco do `Caddyfile.example`.

## Rede e acesso externo

A stack publica o **web** em **`127.0.0.1:8731`** (loopback, não 0.0.0.0). Caddy entra por aí.

> Kestrel (API) está em HTTP, não HTTPS. A imagem oficial não traz certificado TLS, e `HTTPS_PORTS` rebenta logo no arranque com *"No server certificate was specified"*. A TLS é terminada pela Caddy.

## Serviços (containers)

| Serviço | Imagem | Porta interna | Exposto no host? |
| --- | --- | --- | --- |
| `nocturne-postgres-server` | `nocturne-postgres:local` (build de `postgres:17.6` + init script) | 5432 | não |
| `nocturne-api` | `nocturne-api:local` (built from source via `api/Dockerfile`) | 8080 (HTTP) | não (opcional via `NOCTURNE_API_HOST_PORT`) |
| `nocturne-web` | `nocturne-web:local` (built from source via upstream `Dockerfile.web`) | 1612 | `127.0.0.1:8731` |
| `nocturne-watchtower` | `ghcr.io/nicholas-fedor/watchtower:latest` | — | não |

**Connectors** (Dexcom, LibreLinkUp, Glooko, Tidepool, Nightscout, MyFitnessPal, MiniMed CareLink, MyLife, Home Assistant): são *bibliotecas .NET* que correm dentro do próprio `nocturne-api` — não containers separados. Para activar um connector, define as suas env vars no `nocturne-api` (ver `docs/configuration` no upstream).

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

## Configuração da Caddy (padrão loopback)

Este repo assume que a Caddy corre no namespace de rede do host (`network_mode: host` em Docker, ou Caddy nativa) e proxia para `127.0.0.1:<porta>` — o mesmo padrão dos outros sites no Caddyfile.

### 1. Adiciona o bloco ao Caddyfile

Substitui o subdomínio se necessário:

```caddy
nocturne.bfreire.com {
    reverse_proxy 127.0.0.1:8731
}
```

Versão mais completa (com headers e compressão) em [`Caddyfile.example`](./Caddyfile.example).

A Caddy trata automaticamente do certificado Let's Encrypt — desde que o subdomínio resolva para o IP público da máquina e as portas 80/443 estejam acessíveis.

### 2. Recarrega a Caddy

```bash
docker exec <caddy-container> caddy reload --config /etc/caddy/Caddyfile
```

ou reinicia o container.

### 3. Testa

```bash
curl -I https://nocturne.bfreire.com/api/v1/status
```

Deves ver `HTTP/2 200` (ou 401 se o endpoint exigir auth — também conta como sinal de que o reverse proxy funciona).

## Updates automáticos

Watchtower está incluído, mas com `WATCHTOWER_LABEL_ENABLE=true` — só toca em containers com o label `com.centurylinklabs.watchtower.enable=true`. Por defeito `nocturne-api` e `nocturne-web` têm o label, Postgres e o próprio Watchtower ficam estáveis. Polling: 24h.

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
