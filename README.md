# lmtsufape/.github

Workflows reutilizáveis e templates de workflow da organização.

## Deploy no lmtstestes

O servidor de testes `lmtstestes` não tem acesso SSH. As aplicações são atualizadas assim:

```
push ──► GitHub Actions (ubuntu-latest)
           └─ docker-build-push.yml ──► ghcr.io/lmtsufape/<imagem>:<sha>
                                              │
         runner self-hosted da org (label lmtstestes, só conexões de saída)
           └─ deploy-lmtstestes.yml ──► docker compose pull && up -d
```

Logs, restart e console dos containers ficam no Portainer (ver `lmtsufape/infra-lmtstestes`).

### Workflows

| Arquivo | Onde roda | O que faz |
|---|---|---|
| `.github/workflows/docker-build-push.yml` | `ubuntu-latest` | Build de uma imagem e push para o GHCR |
| `.github/workflows/deploy-lmtstestes.yml` | `[self-hosted, lmtstestes]` | Gera `.env`, `compose pull`, `up -d --wait`, verifica URLs, limpa imagens antigas |

### Colocar um sistema no lmtstestes

1. **Porta**: reserve uma porta no registro em `lmtsufape/infra-lmtstestes` (`PORTAS.md`).
2. **Compose de deploy**: crie `deploy/lmtstestes/compose.yml` no repositório:
   - `name: <projeto>` no topo (define o nome de containers e volumes);
   - serviços próprios com `image: ghcr.io/lmtsufape/<projeto>-<serviço>:${IMAGE_TAG}`, sem `build:`;
   - sem bind mounts de arquivos do repositório (o checkout do runner é temporário), use volumes nomeados;
   - credenciais sempre como `${VARIAVEL}`, nunca no arquivo.
3. **Environment**: em *Settings → Environments*, crie `homologacao` com o secret `DEPLOY_ENV_FILE`
   contendo o `.env` (uma variável por linha). `IMAGE_TAG` é adicionada pelo workflow.
4. **Runner**: libere o repositório no runner group *Default* da organização
   (*Settings → Actions → Runner groups*).
5. **Workflow**: em *Actions → New workflow*, use o template **Deploy lmtstestes** e ajuste os nomes.
   O job de deploy precisa de `secrets: inherit`, senão o `DEPLOY_ENV_FILE` chega vazio.

### Rollback

*Actions → (workflow de deploy) → Run workflow*, informando em `image-tag` o SHA de um commit
que já foi implantado. O build é pulado e a imagem antiga é baixada do GHCR.

### Segurança

O runner tem acesso ao Docker do servidor (equivale a root). Por isso:

- só use `runs-on: [self-hosted, lmtstestes]` em jobs disparados por `push` em branch protegida
  ou `workflow_dispatch`, **nunca** em `pull_request`;
- a organização exige aprovação para workflows de PRs de contribuidores externos;
- o runner group *Default* só atende os repositórios selecionados.
