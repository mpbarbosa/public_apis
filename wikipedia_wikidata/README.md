# Wikipedia REST API + Wikidata Action API

Duas APIs da Wikimedia, usadas em conjunto por `agora_na_copa_2026`
(`server.ts`) para montar o cartão de informações de país: resumo e imagem vêm
da Wikipedia; população, área, capital, idioma, governo e moeda vêm do Wikidata.

- Wikipedia REST: <https://pt.wikipedia.org/api/rest_v1/>
- Wikidata Action API: <https://www.wikidata.org/w/api.php>
- Política de uso: <https://foundation.wikimedia.org/wiki/Policy:User-Agent_policy>

---

## 1. Visão geral

| API                  | URL base                                   | Uso                                   |
|----------------------|--------------------------------------------|---------------------------------------|
| Wikipedia REST v1    | `https://{lang}.wikipedia.org/api/rest_v1` | Resumo do artigo, imagens             |
| Wikidata Action API  | `https://www.wikidata.org/w/api.php`       | Entidades, claims, labels, sitelinks  |

`{lang}` é o código da edição (`pt`, `es`, `en`, …). Cada edição é um host
distinto — não existe um host único multilíngue.

---

## 2. Autenticação

**Nenhuma** para leitura. Mas a Wikimedia exige um **User-Agent descritivo**;
agentes genéricos (o padrão do curl, bibliotecas HTTP sem configuração) são
bloqueados com `403`.

Formato recomendado:

```
nome-do-app/versão (URL-do-projeto; email-de-contato)
```

### 2.1 Configuração local

Não há credencial; `.env.example` guarda o User-Agent e a URL base.

```bash
cp wikipedia_wikidata/.env.example wikipedia_wikidata/.env
set -a && . ./wikipedia_wikidata/.env && set +a
```

| Variável              | Descrição                                    |
|-----------------------|----------------------------------------------|
| `WIKIMEDIA_USER_AGENT`| User-Agent descritivo — **obrigatório**      |
| `WIKIDATA_API_BASE`   | `https://www.wikidata.org/w/api.php`         |

---

## 3. Wikipedia REST — resumo de artigo

### `GET /api/rest_v1/page/summary/{title}`

| Parâmetro | Local | Descrição                                             |
|-----------|-------|-------------------------------------------------------|
| `title`   | path  | Título do artigo, **URL-encoded** (`_` no lugar de espaço) |

```bash
curl -s -H "User-Agent: $WIKIMEDIA_USER_AGENT" \
  "https://pt.wikipedia.org/api/rest_v1/page/summary/Brasil"
```

### Campos de resposta

| Campo                 | Tipo   | Descrição                                             |
|-----------------------|--------|-------------------------------------------------------|
| `type`                | string | `standard`, `disambiguation`, `no-extract`            |
| `title`               | string | Título normalizado                                    |
| `displaytitle`        | string | Título com **HTML** — não use cru na interface        |
| `wikibase_item`       | string | QID no Wikidata (ex.: `Q155`) — ponte entre as duas APIs |
| `pageid`              | int    | Id da página na edição                                |
| `description`         | string | Descrição curta de uma linha                          |
| `extract`             | string | Resumo em texto puro                                  |
| `extract_html`        | string | Mesmo resumo em HTML                                  |
| `thumbnail.source`    | string | URL da miniatura                                      |
| `originalimage.source`| string | URL da imagem em tamanho original                     |
| `lang`, `dir`         | string | Idioma e direção do texto                             |
| `timestamp`           | string | Última revisão (ISO 8601)                             |
| `content_urls.desktop.page` | string | URL canônica do artigo                          |

---

## 4. Wikidata Action API — `wbgetentities`

Uma única action cobre os três usos abaixo. Sempre com `format=json`.

### 4.1 Sitelinks — traduzir o título do artigo entre edições

| Parâmetro    | Valor exemplo | Descrição                                  |
|--------------|---------------|--------------------------------------------|
| `action`     | `wbgetentities` |                                          |
| `ids`        | `Q183`        | QID da entidade                            |
| `props`      | `sitelinks`   | Traz os links para as wikis                |
| `sitefilter` | `eswiki`      | Restringe a uma edição (`{lang}wiki`)      |
| `format`     | `json`        |                                            |

```bash
curl -s -H "User-Agent: $WIKIMEDIA_USER_AGENT" \
  "$WIKIDATA_API_BASE?action=wbgetentities&ids=Q183&props=sitelinks&sitefilter=eswiki&format=json"
```

```json
{
  "entities": {
    "Q183": {
      "type": "item", "id": "Q183",
      "sitelinks": {
        "eswiki": { "site": "eswiki", "title": "Alemania", "badges": ["Q17437798"] }
      }
    }
  },
  "success": 1
}
```

### 4.2 Claims — propriedades estruturadas

| Parâmetro   | Valor exemplo | Descrição                      |
|-------------|---------------|--------------------------------|
| `props`     | `claims`      | Traz as declarações da entidade |
| `languages` | `pt`          | Idioma dos rótulos embutidos    |

**Propriedades usadas em `agora_na_copa_2026`**

| Propriedade | Significado          |
|-------------|----------------------|
| `P1082`     | População            |
| `P2046`     | Área                 |
| `P36`       | Capital              |
| `P37`       | Idioma oficial       |
| `P122`      | Forma de governo     |
| `P38`       | Moeda                |

Uma propriedade pode ter **várias claims** (valores históricos). A regra usada
no projeto: *rank* `preferred` > claim sem `end time` > última claim.

### 4.3 Labels em lote — resolver QIDs para nomes

`P36`, `P37`, `P122` e `P38` devolvem **QIDs**, não texto. Resolva todos em uma
única chamada:

| Parâmetro   | Valor exemplo       | Descrição                             |
|-------------|---------------------|---------------------------------------|
| `ids`       | `Q64\|Q188`         | Vários QIDs separados por `\|` (`%7C`) |
| `props`     | `labels`            |                                       |
| `languages` | `pt\|en`            | Ordem de preferência do rótulo        |

```bash
curl -s -H "User-Agent: $WIKIMEDIA_USER_AGENT" \
  "$WIKIDATA_API_BASE?action=wbgetentities&ids=Q64%7CQ188&languages=pt%7Cen&props=labels&format=json"
```

```json
{
  "entities": {
    "Q64":  { "type": "item", "id": "Q64",  "labels": { "pt": { "language": "pt", "value": "Berlim" }, "en": { "language": "en", "value": "Berlin" } } },
    "Q188": { "type": "item", "id": "Q188", "labels": { "pt": { "language": "pt", "value": "alemão" }, "en": { "language": "en", "value": "German" } } }
  },
  "success": 1
}
```

---

## 5. Resumo dos endpoints

| Método | URL                                                        | Descrição                    |
|--------|------------------------------------------------------------|------------------------------|
| GET    | `https://{lang}.wikipedia.org/api/rest_v1/page/summary/{title}` | Resumo + imagens do artigo |
| GET    | `…/w/api.php?action=wbgetentities&props=sitelinks`          | Título em outra edição       |
| GET    | `…/w/api.php?action=wbgetentities&props=claims`             | Propriedades estruturadas    |
| GET    | `…/w/api.php?action=wbgetentities&props=labels`             | Rótulos de QIDs (em lote)    |

### Fluxo típico (cartão de país localizado)

1. Título em pt conhecido + QID conhecido.
2. Locale ≠ pt → `props=sitelinks&sitefilter={lang}wiki` para achar o título traduzido.
3. `page/summary/{título}` na edição `{lang}` → resumo e imagem.
4. `props=claims` no QID → P1082, P2046, P36, P37, P122, P38.
5. `props=labels` em lote nos QIDs dessas claims → nomes legíveis.

---

## 6. Limites de uso

- User-Agent descritivo é **obrigatório** (política da Wikimedia Foundation).
- Sem cota fixa publicada para leitura anônima; a recomendação é serializar as
  requisições e não paralelizar agressivamente.
- Os dados mudam raramente — `agora_na_copa_2026` cacheia por **24 horas**.
- Conteúdo sob CC BY-SA (Wikipedia) e CC0 (Wikidata); atribuição conforme a licença.

---

## 7. Comportamento observado (verificado em campo)

Verificado em 01/09/2026 com chamadas reais:

| # | Observação |
|---|------------|
| 1 | `HTTP 200` sem autenticação, desde que o User-Agent seja descritivo. Um UA genérico leva a `403`. |
| 2 | `wikibase_item` no resumo da Wikipedia é a ponte para o Wikidata — evita uma busca por título. |
| 3 | `displaytitle` vem com marcação HTML (`<span lang="pt" dir="ltr">…`); use `title` para texto puro. |
| 4 | Sitelinks confirmam a tradução do título: `Q183` + `sitefilter=eswiki` → `"Alemania"`. |
| 5 | O separador de múltiplos `ids` é `\|`, que precisa ir como `%7C` na query. |
| 6 | Nem todo QID tem rótulo em pt (ex.: moedas) — encadeie o fallback `{locale} → en → pt`. |
| 7 | Uma edição pode não ter artigo para a entidade: `sitelinks` volta vazio. Trate como "sem tradução" e caia na edição padrão. |
| 8 | `success: 1` acompanha as respostas do Wikidata; ausência dele indica erro mesmo com HTTP 200. |

---

*Documento compilado a partir da documentação oficial da Wikimedia e validado
com chamadas reais. Integração de referência: `fetchCountryInfo` /
`localizedWikipediaArticle` em `server.ts` de `agora_na_copa_2026`.*
