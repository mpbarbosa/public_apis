# Reddit API — OAuth2 (app-only)

API de leitura do Reddit. Usada em `agora_na_copa_2026` (`reddit-core.ts`) para
hidratar posts curados com contagem de votos e comentários ao vivo.

- Documentação: <https://www.reddit.com/dev/api/>
- Regras de uso: <https://support.reddithelp.com/hc/en-us/articles/16160319875092>
- Registro de aplicativos: <https://www.reddit.com/prefs/apps>

---

## 1. Visão geral

Desde 2023 o Reddit **encerrou o acesso anônimo gratuito**: toda chamada exige
um token OAuth2. Para aplicações sem usuário logado, o fluxo correto é
**app-only** (`grant_type=client_credentials`).

### URLs base

| Finalidade          | URL                                          |
|---------------------|----------------------------------------------|
| Emissão de token    | `https://www.reddit.com/api/v1/access_token` |
| Chamadas de dados   | `https://oauth.reddit.com`                   |

> ⚠️ São **hosts diferentes**. O token sai de `www.reddit.com`; os dados vêm de
> `oauth.reddit.com`. Chamar `www.reddit.com/...json` com o token não funciona.

---

## 2. Autenticação

### 2.1 Como obter as credenciais

1. Acesse <https://www.reddit.com/prefs/apps> logado na sua conta.
2. Clique em **"create another app..."** e escolha o tipo:
   - **script** — uso pessoal, acesso à própria conta;
   - **web app** — servidor com redirect URI;
   - **installed app** — cliente sem segredo.
3. Preencha nome, descrição e *redirect uri* (para app-only qualquer URL válida
   serve, ex.: `http://localhost:3000`).
4. Após criar, a página mostra:
   - **client id** — a string sob o nome do app (logo abaixo de "personal use script");
   - **client secret** — o campo `secret`.

### 2.2 Configuração local

As credenciais ficam em `reddit/.env`, **fora do controle de versão**
(`.gitignore` deste diretório). Use `.env.example` como modelo:

```bash
cp reddit/.env.example reddit/.env   # e preencha id/secret
chmod 600 reddit/.env
set -a && . ./reddit/.env && set +a
```

| Variável               | Descrição                                            |
|------------------------|------------------------------------------------------|
| `REDDIT_CLIENT_ID`     | Client id do aplicativo                              |
| `REDDIT_CLIENT_SECRET` | Client secret do aplicativo                          |
| `REDDIT_USER_AGENT`    | User-Agent descritivo — **obrigatório**              |

### 2.3 `POST /api/v1/access_token`

Autenticação **HTTP Basic** com `client_id:client_secret` em base64.

| Cabeçalho       | Valor                                             |
|-----------------|---------------------------------------------------|
| `Authorization` | `Basic base64(client_id:client_secret)`           |
| `Content-Type`  | `application/x-www-form-urlencoded`               |
| `User-Agent`    | Descritivo (ver seção 5)                          |

Corpo: `grant_type=client_credentials`

```bash
curl -s -X POST \
  -u "$REDDIT_CLIENT_ID:$REDDIT_CLIENT_SECRET" \
  -H "User-Agent: $REDDIT_USER_AGENT" \
  -d "grant_type=client_credentials" \
  https://www.reddit.com/api/v1/access_token
```

**Resposta**

```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "bearer",
  "expires_in": 86400,
  "scope": "*"
}
```

| Campo          | Tipo   | Descrição                                          |
|----------------|--------|----------------------------------------------------|
| `access_token` | string | Token bearer                                       |
| `token_type`   | string | Sempre `bearer`                                    |
| `expires_in`   | int    | Validade em segundos (tipicamente `86400` = 24 h)  |
| `scope`        | string | Escopos concedidos                                 |

As chamadas seguintes usam `Authorization: Bearer <access_token>`.

---

## 3. Endpoints de leitura

### `GET /api/info`

Hidrata vários posts em uma única chamada, por *fullname*.

| Parâmetro | Tipo   | Descrição                                                  |
|-----------|--------|------------------------------------------------------------|
| `id`      | string | Lista de fullnames separados por vírgula (`t3_<id>,t3_<id>`) |
| `url`     | string | Alternativa: busca pelo URL do post                        |

**Fullnames** — prefixo por tipo de objeto:

| Prefixo | Objeto     |
|---------|------------|
| `t1_`   | Comentário |
| `t3_`   | Post/link  |
| `t5_`   | Subreddit  |

O id base-36 de um post sai do permalink: `/r/<sub>/comments/<id>/<slug>/`.

```bash
curl -s -H "Authorization: Bearer $TOKEN" -H "User-Agent: $REDDIT_USER_AGENT" \
  "https://oauth.reddit.com/api/info?id=t3_1abcdef,t3_1ghijkl"
```

### `GET /r/{subreddit}/hot` · `/new` · `/top`

Listagem de um subreddit.

| Parâmetro | Tipo   | Descrição                                            |
|-----------|--------|------------------------------------------------------|
| `limit`   | int    | 1–100 (padrão 25)                                    |
| `after`   | string | Fullname para paginação                              |
| `t`       | string | Só em `/top`: `hour`, `day`, `week`, `month`, `year`, `all` |

### `GET /search` · `GET /r/{subreddit}/search`

| Parâmetro     | Tipo   | Descrição                                        |
|---------------|--------|--------------------------------------------------|
| `q`           | string | Termo de busca                                   |
| `sort`        | string | `relevance`, `hot`, `top`, `new`, `comments`     |
| `restrict_sr` | bool   | Restringe ao subreddit da rota                   |
| `limit`       | int    | 1–100                                            |

### Estrutura de resposta (Listing)

```json
{
  "kind": "Listing",
  "data": {
    "after": "t3_1ghijkl",
    "before": null,
    "children": [
      {
        "kind": "t3",
        "data": {
          "id": "1abcdef",
          "title": "...",
          "subreddit": "worldcup",
          "author": "...",
          "score": 1234,
          "num_comments": 89,
          "permalink": "/r/worldcup/comments/1abcdef/...",
          "url": "https://...",
          "created_utc": 1788267506,
          "over_18": false,
          "thumbnail": "https://..."
        }
      }
    ]
  }
}
```

| Campo                  | Tipo    | Descrição                                       |
|------------------------|---------|-------------------------------------------------|
| `kind`                 | string  | `Listing` no envelope; `t3` em cada filho       |
| `data.after`/`before`  | string  | Cursores de paginação (fullnames)               |
| `data.children[]`      | array   | Objetos do listing                              |
| `…data.id`             | string  | Id base-36 do post                              |
| `…data.title`          | string  | Título                                          |
| `…data.subreddit`      | string  | Subreddit, sem o prefixo `r/`                   |
| `…data.author`         | string  | Autor (`[deleted]` quando removido)             |
| `…data.score`          | int     | Votos líquidos (upvotes − downvotes)            |
| `…data.num_comments`   | int     | Número de comentários                           |
| `…data.permalink`      | string  | Caminho relativo — prefixe `https://www.reddit.com` |
| `…data.created_utc`    | double  | Epoch UTC em **segundos**                       |
| `…data.over_18`        | bool    | Conteúdo adulto                                 |

---

## 4. Resumo dos endpoints

| Método | Caminho                        | Host                 | Descrição              |
|--------|--------------------------------|----------------------|------------------------|
| POST   | `/api/v1/access_token`         | `www.reddit.com`     | Emissão do token       |
| GET    | `/api/info`                    | `oauth.reddit.com`   | Hidratar posts por id  |
| GET    | `/r/{sub}/hot` `/new` `/top`   | `oauth.reddit.com`   | Listagens              |
| GET    | `/search`                      | `oauth.reddit.com`   | Busca                  |
| GET    | `/comments/{id}`               | `oauth.reddit.com`   | Post + comentários     |

---

## 5. Limites de uso

- **User-Agent descritivo é obrigatório.** O formato recomendado é
  `<plataforma>:<id do app>:<versão> (by /u/<usuário>)`. Agentes genéricos ou
  ausentes recebem `429`.
- Cota do app-only: **100 requisições por minuto** por client id
  (média em janela de 10 min). Os cabeçalhos `x-ratelimit-remaining`,
  `x-ratelimit-used` e `x-ratelimit-reset` acompanham cada resposta.
- O token dura 24 h — reaproveite-o em cache; não peça um por requisição.
- Uso comercial ou volume alto exige contato com o Reddit (termos de 2023).

---

## 6. Comportamento observado (verificado em campo)

Verificado em 01/09/2026:

| # | Observação |
|---|------------|
| 1 | `POST /api/v1/access_token` **sem** Basic auth retorna `401` — confirmado. |
| 2 | `GET oauth.reddit.com/api/info` sem `Authorization` retorna `403` (não `401`) — trate os dois como "reautenticar". |
| 3 | O token e os dados vivem em hosts distintos; errar o host é a falha mais comum. |
| 4 | `expires_in` é reportado pelo Reddit e pode variar — não fixe 86400 no código; use o valor da resposta (o `reddit-core.ts` cai para `3600` quando o campo falta). |
| 5 | `created_utc` está em **segundos**, não milissegundos — multiplique por 1000 antes de `new Date()`. |
| 6 | `permalink` é relativo; concatene com `https://www.reddit.com`. |
| 7 | Sem credenciais a integração deve degradar, não quebrar: `agora_na_copa_2026` serve um seed curado com `source: "fallback"`. |

---

*Documento compilado a partir da documentação oficial do Reddit e validado com
chamadas reais (fluxo não autenticado). Integração de referência:
`reddit-core.ts` em `agora_na_copa_2026`.*
