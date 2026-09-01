# Google Trends — RPC `batchexecute` (não oficial)

Fonte dos assuntos em alta ("Trending now", últimas 24 h). Usada em
`agora_na_copa_2026` (`trends-core.ts`) para a aba de tendências.

- Interface pública: <https://trends.google.com/trending>

> ⚠️ **API não oficial.** Este é o RPC interno que alimenta a interface do
> Google Trends — o mesmo conjunto de dados que a UI exporta em CSV. Não há
> contrato público, SLA ou versionamento: o Google pode alterar o formato sem
> aviso. Todo parser deve degradar para lista vazia em vez de lançar exceção.

---

## 1. Visão geral

O Google não publica uma API REST para o Trends. O feed `/trending/rss`
existe, mas devolve o conjunto **"Daily Search Trends"** — bem menor e ordenado
de forma diferente. Para os mesmos dados da tela "Em alta agora", usa-se o
endpoint `batchexecute` com o `rpcid` **`i0OFE`**.

### URL

```
https://trends.google.com/_/TrendsUi/data/batchexecute?rpcids=i0OFE&source-path=%2Ftrending&hl=pt-BR
```

| Parâmetro     | Valor                | Descrição                              |
|---------------|----------------------|----------------------------------------|
| `rpcids`      | `i0OFE`              | Identificador do RPC "trending now"    |
| `source-path` | `%2Ftrending`        | Página de origem (a UI envia isto)     |
| `hl`          | `pt-BR`              | Idioma da interface/resposta           |

---

## 2. Autenticação

**Nenhuma.** Sem chave, sem cookie, sem cadastro.

### 2.1 Configuração local

Não há credencial a proteger; `.env.example` guarda apenas URL, país e idioma.

```bash
cp google_trends/.env.example google_trends/.env
set -a && . ./google_trends/.env && set +a
```

---

## 3. Requisição

### `POST /_/TrendsUi/data/batchexecute`

| Cabeçalho      | Valor                                                 |
|----------------|-------------------------------------------------------|
| `Content-Type` | `application/x-www-form-urlencoded;charset=UTF-8`      |

O corpo é o campo `f.req`, urlencoded, contendo um JSON aninhado:

```
f.req = urlencode( JSON([[[ "i0OFE", JSON([null,null,geo,0,"",hours,1]) ]]]) )
```

### Argumentos internos

| Posição | Valor exemplo | Descrição                                    |
|---------|---------------|----------------------------------------------|
| 0, 1    | `null`        | Não usados                                    |
| 2       | `"BR"`        | Código do país (ISO-2)                        |
| 3       | `0`           | Reservado                                     |
| 4       | `""`          | Reservado                                     |
| 5       | `24`          | Janela em horas (24 = últimas 24 h)           |
| 6       | `1`           | Reservado                                     |

**Exemplo**

```bash
BODY="f.req=$(python3 -c "
import json,urllib.parse
inner = json.dumps([None, None, 'BR', 0, '', 24, 1])
print(urllib.parse.quote(json.dumps([[['i0OFE', inner]]])))")"

curl -s -X POST \
  -H "Content-Type: application/x-www-form-urlencoded;charset=UTF-8" \
  --data "$BODY" \
  "https://trends.google.com/_/TrendsUi/data/batchexecute?rpcids=i0OFE&source-path=%2Ftrending&hl=pt-BR"
```

---

## 4. Resposta

O corpo **não é JSON puro**: vem prefixado por `)]}'` (anti-JSON-hijacking) e
carrega JSON serializado dentro de strings.

```
)]}'

[["wrb.fr","i0OFE","[null,[[\"remo x coritiba\",null,\"BR\",[1788222000],null,null,500000,null,1000,[...],[17],...]]]"]]
```

### Como fazer o parse

1. Remova o prefixo `)]}'` e espaços iniciais.
2. `JSON.parse` no restante → array de linhas.
3. Localize a linha em que `row[1] === "i0OFE"` e `row[2]` é string.
4. `JSON.parse(row[2])` → `[null, [entrada, entrada, ...]]`.
5. As entradas ficam em `inner[1]`.

### Estrutura de cada entrada

| Índice | Tipo     | Descrição                                              |
|--------|----------|--------------------------------------------------------|
| 0      | string   | Título do assunto em alta                              |
| 2      | string   | Código do país                                         |
| 3      | int[]    | `[startTs]` — epoch UTC em **segundos** do início       |
| 6      | int      | Volume aproximado de buscas                            |
| 8      | int      | Crescimento percentual                                 |
| 9      | string[] | Consultas relacionadas                                 |
| 10     | int[]    | Códigos de categoria                                   |

Os índices 1, 4, 5, 7 vêm nulos nas amostras coletadas.

### Categorias

| Código | Categoria |
|--------|-----------|
| `17`   | Esportes  |

### Formatação de volume (estilo pt-BR)

| Faixa           | Formato   |
|-----------------|-----------|
| ≥ 1.000.000     | `2 mi+`   |
| ≥ 1.000         | `500 mil+`|
| < 1.000         | `200+`    |

---

## 5. Limites de uso

- Não há cota publicada. Por ser endpoint interno, chamadas frequentes tendem a
  receber `429` ou desafio. `agora_na_copa_2026` cacheia por **20 minutos**.
- Os dados do "Trending now" mudam poucas vezes por hora; não vale consultar
  com mais frequência que isso.
- Sem termos de uso para consumo programático — avalie o risco antes de usar em
  produção e mantenha sempre um caminho de fallback.

---

## 6. Comportamento observado (verificado em campo)

Verificado em 01/09/2026 com chamada real (`geo=BR`, `hours=24`):

| # | Observação |
|---|------------|
| 1 | `HTTP 200` sem autenticação nem cookie — confirmado. |
| 2 | O corpo começa com `)]}'` seguido de **duas quebras de linha**; sem removê-las o `JSON.parse` falha. |
| 3 | Os dados úteis são JSON **dentro de string** (`row[2]`) — exige um segundo `JSON.parse`. |
| 4 | O timestamp em `entry[3]` é `[epochSegundos]` — um array de um elemento, não um escalar. |
| 5 | O volume vem arredondado pelo Google (ex.: `500000`), não é contagem exata. |
| 6 | A ordem das entradas **é** o ranking do Google; preserve-a em vez de reordenar. |
| 7 | O formato não tem contrato: valide o shape a cada campo e retorne `[]` em caso de divergência, como faz `parseGoogleTrendsBatch`. |

---

*Documento compilado por engenharia reversa do RPC público do Google Trends e
validado com chamadas reais. Integração de referência: `trends-core.ts` em
`agora_na_copa_2026`. Não há fonte oficial a citar.*
