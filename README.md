# nocturne-docker

Stack Docker pronta a puxar pelo **Portainer** para correr o [Nocturne](https://github.com/nightscout/nocturne) — a reescrita em .NET 10 da API do Nightscout — usando as imagens oficiais publicadas no GHCR pela CI do projecto upstream. Nenhum código do Nocturne é alterado: só wrap.

> Imagens consumidas: `ghcr.io/nightscout/nocturne/nocturne-api`. Base de dados: PostgreSQL 17.6 com os três roles oficiais (`nocturne_migrator`, `nocturne_app`, `nocturne_web`) provisionados pelo script `pg-init/00-init.sh` (cópia *byte-a-byte* de [`docs/postgres/container-init/00-init.sh`](https://github.com/nightscout/nocturne/blob/main/docs/postgres/container-init/00-init.sh)).

## Conteúdo do repo

| Ficheiro | O que faz |
| --- | --- |
| `docker-compose.yml` | Stack: `postgres` + `nocturne-api` + `watchtower` (com label scope). |
| `.env.example` | Template das variáveis. Copia para `.env` e preenche. |
| `pg-init/00-init.sh` | Cria os três roles PostgreSQL na primeira arranque do container Postgres. |
| `.gitignore` | Garante que `.env` nunca vai para o repo. |

## Porta exposta

A stack publica a API em **`https://<host>:8731`** no host (mapeada para `8443/HTTPS` dentro do container Kestrel). Para mudar, edita `NOCTURNE_HOST_PORT` no `.env`.

> O Nocturne fala HTTPS internamente com cert self-signed do container. Para uso em produção real, mete um reverse proxy (Traefik / Caddy / nginx) à frente e termina TLS lá.

## Como subir no Portainer (Stack from Git)

1. **Portainer → Stacks → Add stack → Repository**.
2. **Repository URL**: `https://github.com/bfreire/nocturne-docker` (depois de fazeres o push abaixo).
3. **Compose path**: `docker-compose.yml`.
4. **Environment variables**: copia as chaves do `.env.example` e preenche os valores. **Não** uses o ficheiro `.env` por upload — Portainer prefere envs colados no formulário (assim ficam no Portainer e não no repo).
5. **Deploy the stack**.

Notas:

- O `pg-init/00-init.sh` é montado via *bind mount relativo* (`./pg-init`). Quando o Portainer faz git pull, o ficheiro vai junto e o mount funciona — desde que o agent Portainer corra no mesmo host onde o repo é clonado (default).
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
