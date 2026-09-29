# TSE — Tribunal Superior Eleitoral

Dados eleitorais públicos do Brasil: candidaturas, bens declarados, prestação
de contas, eleitorado e resultados. **Gratuitos e sem chave**, mas espalhados
por três serviços independentes — e todos atrás de um bloqueio anti-bot
(Akamai) que recusa clientes HTTP comuns (ver §2).

- Portal de dados abertos: <https://dadosabertos.tse.jus.br>
- Consulta de candidaturas: <https://divulgacandcontas.tse.jus.br>
- Resultados: <https://resultados.tse.jus.br>
- Licença dos dados abertos: CC BY 4.0 (`license_id: cc-by`)

---

## 1. Visão geral

| Serviço | Host | O que entrega | Documentação oficial |
|---------|------|---------------|----------------------|
| **Dados Abertos** | `dadosabertos.tse.jus.br` | Catálogo CKAN 2.11.5 de arquivos em lote (CSV em ZIP) servidos por `cdn.tse.jus.br` | API CKAN padrão |
| **DivulgaCandContas** | `divulgacandcontas.tse.jus.br` | JSON de candidaturas, bens, arquivos e prestação de contas por candidato | **Nenhuma** — é o backend do site |
| **Resultados** | `resultados.tse.jus.br` | JSON da apuração em tempo real | Nenhuma; fora do ar em 28/09/2026 (§5) |

Escolha pelo uso:

- **Análise histórica / em lote** → Dados Abertos (candidatos desde 1933).
- **Um candidato, dados atuais** → DivulgaCandContas.
- **Apuração ao vivo** → Resultados (quando voltar).

### URLs base

```
https://dadosabertos.tse.jus.br/api/3/action
https://divulgacandcontas.tse.jus.br/divulga/rest/v1
https://resultados.tse.jus.br/oficial
```

Todas as respostas de API são **JSON**. Somente `GET`.

---

## 2. Autenticação e acesso

**Nenhuma autenticação.** Não há token, chave ou cadastro.

### ⚠️ Bloqueio anti-bot (Akamai)

Todos os hosts `*.tse.jus.br` — inclusive `www` e o CDN de arquivos —
respondem **`403 Access Denied`** (página HTML da Akamai, com `Reference #…`)
a clientes HTTP de linha de comando. O bloqueio **não é por IP nem por
User-Agent**: foi reproduzido a partir de IP residencial brasileiro
(Telefônica, São Paulo) e com User-Agent de Chrome. As mesmas URLs respondem
`200` quando abertas num navegador real.

Consequências:

- `curl`, `requests`, `fetch` do Node etc. **não funcionam** de forma
  confiável.
- As APIs **não enviam `Access-Control-Allow-Origin`**, então uma página em
  outro domínio também não consegue chamá-las pelo navegador.
- O acesso que funciona hoje é **navegar diretamente** às URLs num navegador
  (ou automação de navegador conduzida por um usuário).

Alternativas legítimas para uso em servidor:

- Solicitar liberação ao TSE pelo canal indicado nos metadados dos conjuntos
  (formulário da Assessoria de Informação ao Cidadão).
- Usar um espelho que republica os dados do TSE, como a
  [Base dos Dados](https://basedosdados.org) (BigQuery).

### 2.1 Configuração local

Não há credencial a proteger; `.env.example` guarda apenas as URLs base.

```bash
cp tse/.env.example tse/.env
set -a && . ./tse/.env && set +a
```

---

## 3. Dados Abertos (CKAN)

API CKAN padrão (Action API v3). Toda resposta segue o envelope:

```json
{ "help": "…/help_show?name=<ação>", "success": true, "result": … }
```

Em erro, `success: false` e um objeto `error`
(ex.: `404` → `{"__type": "Not Found Error", "message": "Não encontrado"}`).

### 3.1 Endpoints úteis

| Ação | Uso |
|------|-----|
| `GET /status_show` | Versão do CKAN e extensões ativas |
| `GET /package_list` | Nomes de todos os conjuntos (182 em 28/09/2026) |
| `GET /package_search?q=<termo>&rows=<n>&start=<n>` | Busca textual paginada |
| `GET /package_show?id=<nome>` | Metadados e arquivos (`resources`) de um conjunto |
| `GET /current_package_list_with_resources?limit=<n>` | Todos os conjuntos com seus arquivos, numa chamada |
| `GET /organization_list` | Áreas gestoras (`tse-agel`, `tse-sti`, …) |

### 3.2 Convenção de nomes dos conjuntos

`<tema>-<ano>`, por exemplo `candidatos-2026`, `resultados-2024`,
`eleitorado-2026`. Temas principais e quantidade de anos publicados:

| Tema | Conjuntos |
|------|-----------|
| `resultados` | 61 |
| `candidatos` | 36 |
| `eleitorado` | 17 |
| `prestacao-de-contas-partidarias` | 13 |
| `prestacao-de-contas-eleitorais` | 12 |
| `pesquisas-eleitorais` | 8 |
| `comparecimento-e-abstencao` | 7 |
| `denuncias-eleitorais` | 7 |

Conjuntos de 2026 já publicados: `candidatos-2026`, `eleitorado-2026`,
`mesarias-mesarios-e-funcoes-especiais-2026`, `pesquisas-eleitorais-2026`,
`prestacao-de-contas-eleitorais-2026`, `prestacao-de-contas-partidarias-2026`,
`processual-2026`.

### 3.3 Arquivos (`resources`)

Cada `resource` aponta para um ZIP em `cdn.tse.jus.br`:

```
https://cdn.tse.jus.br/estatistica/sead/odsele/<tema>/<tema>_<ano>.zip
```

Exemplos:

| Conjunto | Arquivo | Caminho no CDN |
|----------|---------|----------------|
| `candidatos-2026` | Candidatos | `/estatistica/sead/odsele/consulta_cand/consulta_cand_2026.zip` |
| `candidatos-2026` | Bens de candidatos | `/estatistica/sead/odsele/bem_candidato/bem_candidato_2026.zip` |
| `resultados-2024` | Votação nominal por município e zona | `/estatistica/sead/odsele/votacao_candidato_munzona/votacao_candidato_munzona_2024.zip` |
| `resultados-2024` | Detalhe da apuração por seção | `/estatistica/sead/odsele/detalhe_votacao_secao/detalhe_votacao_secao_2024.zip` |

`candidatos-2026` tem 92 arquivos: 8 CSVs nacionais mais, **por UF (incluindo
`BR`)**, fotos, propostas de governo e certidões criminais.

Campos relevantes de cada `resource`:

| Campo | Observação |
|-------|------------|
| `url` | Link direto para o ZIP no CDN |
| `format` | `CSV`, `TXT`, `PDF`, `JPEG`, `ZIP` — ou **vazio** em ~30% dos arquivos |
| `size` | Sempre `null` |
| `last_modified` | Data de cadastro no catálogo, **não** da geração do arquivo |
| `datastore_active` | Sempre `false` |

A frequência declarada fica em `extras` do conjunto
(`"Frequência de atualização": "4 vezes ao dia"` em `candidatos-2026`).

---

## 4. DivulgaCandContas (REST não documentada)

Backend do site de consulta de candidaturas. Rápido (30–90 ms), mas **sem
contrato**: as rotas usam parâmetros posicionais e podem mudar sem aviso.

### 4.1 `GET /eleicao/ordinarias`

Lista as eleições ordinárias. O `id` é a chave de todas as outras rotas.

| `id` | `ano` | `nomeEleicao` | `dataEleicao` | `tipoAbrangencia` |
|------|-------|---------------|---------------|-------------------|
| `20322002026` | 2026 | Eleição Geral Federal 2026 | `2026-10-04` | `F` |
| `2045202024` | 2024 | Eleições Municipais 2024 | `2024-10-06` | `M` |
| `2040602022` | 2022 | Eleição Geral Federal 2022 | `2022-10-02` | `F` |
| `2022802018` | 2018 | Eleição Geral Federal 2018 | `2018-10-07` | `F` |

`tipoAbrangencia`: `F` = federal/geral, `M` = municipal. A lista vai até 2004;
os ids antigos são curtos (`2` para 2016, `680` para 2014).

### 4.2 `GET /eleicao/listar/municipios/{idEleicao}/{UE}/cargos`

Cargos disputados numa unidade eleitoral, com a contagem de candidaturas.
`UE` é a UF (`SP`, `DF`…) ou `BR` para cargos nacionais.

| `codigo` | Cargo | Onde |
|----------|-------|------|
| 1 | Presidente | `BR` |
| 2 | Vice-presidente | `BR` |
| 3 | Governador | UF |
| 4 | Vice-governador | UF |
| 5 | Senador | UF |
| 6 | Deputado Federal | UF |
| 7 | Deputado Estadual | UF (exceto DF) |
| 8 | Deputado Distrital | `DF` |
| 9 | 1º Suplente (senado) | UF |
| 10 | 2º Suplente (senado) | UF |

### 4.3 `GET /candidatura/listar/{ano}/{UE}/{idEleicao}/{cargo}/candidatos`

Lista resumida de candidatos de um cargo.

```
/candidatura/listar/2026/SP/20322002026/3/candidatos
```

Resposta: `{ unidadeEleitoral, cargo, candidatos: [...] }`. Campos preenchidos
em cada candidato:

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `id` | number | Id do candidato (usado no detalhe e na foto) |
| `nomeUrna` / `nomeCompleto` | string | Nome na urna / nome civil |
| `numero` | number | Número de urna |
| `descricaoSituacao` | string | Situação do registro (ex.: `Deferido`) |
| `descricaoTotalizacao` | string | Ex.: `Concorrendo` |
| `candidatoApto` | boolean | Apto a receber votos |
| `st_REELEICAO` | boolean | Concorre à reeleição |
| `partido` | object | `{ sigla, numero, nome }` — `numero` vem `0`, `nome` vem `null` |
| `nomeColigacao` | string | Federação/coligação |
| `gastoCampanha` | number | Gasto declarado |
| `tituloEleitor` | string | **Dado pessoal** — ver §7 |

Os demais campos (sexo, nascimento, cor/raça, ocupação…) vêm `null` na
listagem; use o detalhe.

### 4.4 `GET /candidatura/buscar/{ano}/{UE}/{idEleicao}/candidato/{idCandidato}`

Ficha completa. Além dos campos da listagem, preenche:

| Campo | Descrição |
|-------|-----------|
| `descricaoSexo`, `descricaoCorRaca`, `descricaoEstadoCivil`, `grauInstrucao`, `ocupacao` | Perfil |
| `dataDeNascimento` | `YYYY-MM-DD` |
| `cpf` | **CPF completo, sem máscara** — ver §7 |
| `bens[]` | `{ ordem, descricao, descricaoDeTipoDeBem, valor, dataUltimaAtualizacao }` |
| `totalDeBens` | Soma dos bens |
| `vices[]` | Vice/suplentes da chapa |
| `arquivos[]` | `{ idArquivo, nome, tipo, codTipo, url }` — plano de governo (`codTipo 5`), certidões (`11`–`14`) etc. |
| `eleicoesAnteriores[]` | `{ nrAno, cargo, local, partido, situacaoTotalizacao, … }` |
| `st_MOTIVO_*` | Flags de indeferimento (ficha limpa, abuso de poder, compra de voto…) |
| `fotoUrl` | URL da foto (§4.6) |
| `dataUltimaAtualizacao` | `YYYY-MM-DD HH:mm` |

### 4.5 `GET /prestador/consulta/{idEleicao}/{ano}/{UE}/{cargo}/{partido}/{numero}/{idCandidato}`

Prestação de contas de campanha. O segmento `{partido}` aceitou o número do
candidato no teste (a listagem devolve `partido.numero = 0`).

| Campo | Descrição |
|-------|-----------|
| `dataUltimaAtualizacaoContas` | `DD/MM/YYYY` |
| `dadosConsolidados` | Totais, quantidades e percentuais de receita por origem (PF, PJ, partidos, FEFC, internet, próprios…) |
| `despesas` | `totalDespesasContratadas`, `totalDespesasPagas`, `valorLimiteDeGastos`, `limiteDeGasto1T/2T` |
| `rankingDoadores[]` | `{ nome, cpfCnpj, valor, qntd, stFinanciamentoColetivo }` |
| `rankingFornecedores[]` | Maiores fornecedores |
| `concentracaoDespesas` | Despesas por categoria |
| `dividaCampanha`, `sobraFinanceira*` | Dívidas e sobras |

### 4.6 `GET /divulga/rest/arquivo/img/{idEleicao}/{idCandidato}/{UE}`

Foto do candidato (`image/png`). Fica **fora** de `/rest/v1`.

---

## 5. Resultados

Em anos anteriores o aplicativo Resultados servia JSONs estáticos em
`/oficial/ele<ano>/…` (configuração, dados simplificados por abrangência).

Em **28/09/2026** todas as rotas — inclusive as de 2022 e 2024 — devolvem
`404` com a página HTML *"Resultados | Nova versão em breve"*, que se recarrega
a cada 5 minutos. O formato de 2026 **não pôde ser verificado**; revalidar
após a publicação da nova versão (1º turno em 04/10/2026) e não presumir
compatibilidade com 2022/2024.

Para resultados consolidados, use os conjuntos `resultados-<ano>` do portal
de dados abertos (§3).

---

## 6. Limites de uso

Não há limite de requisições documentado em nenhum dos serviços. Observado:

- `cache-control`: `max-age=3600` no CKAN, `max-age=1700` nas rotas do
  DivulgaCandContas e `max-age≈270` nas fotos.
- O bloqueio da Akamai (§2) é o limite prático; volume alto de chamadas pode
  endurecê-lo. Prefira os ZIPs em lote a varrer candidatos um a um.

---

## 7. Comportamento observado (verificado em campo)

Verificado em 28/09/2026 com chamadas reais:

| # | Observação |
|---|------------|
| 1 | `curl` recebe `403 Access Denied` (Akamai) em todos os hosts, inclusive de IP residencial brasileiro e com User-Agent de navegador. No navegador, `200`. |
| 2 | Nenhuma API envia `Access-Control-Allow-Origin`: não dá para consumi-las de uma página em outro domínio. |
| 3 | O CKAN tem a extensão `datastore` ativa, mas **nenhum** dos 5.316 arquivos está carregado nela — não há consulta por linha, só download do ZIP. |
| 4 | `size` é sempre `null` e ~1.550 arquivos não têm `format` declarado. |
| 5 | `last_modified` do CKAN é a data de cadastro do arquivo no catálogo; o ZIP no CDN é regerado com frequência maior ("4 vezes ao dia" em `candidatos-2026`). |
| 6 | O DivulgaCandContas não valida parâmetros: cargo inexistente (`999`) devolve `200` com `candidatos: []`, não `404`. |
| 7 | Na listagem, `partido.numero` vem `0`, `cargo.contagem` vem `0` e o objeto `eleicao` vem quase todo `null`; os valores reais estão no detalhe e na rota de cargos. |
| 8 | Três formatos de data na mesma API: `YYYY-MM-DD` (eleição), `YYYY-MM-DD HH:mm` (candidato) e `DD/MM/YYYY` (contas). |
| 9 | **Dados pessoais:** o detalhe expõe o **CPF completo sem máscara**, título de eleitor e data de nascimento; a listagem expõe o título; `rankingDoadores` expõe CPF/CNPJ completo dos doadores. Quem armazenar isso responde pelo tratamento (LGPD) — descarte esses campos na ingestão se não forem necessários. |
| 10 | `resultados.tse.jus.br` está fora do ar para todas as eleições, com página de "nova versão em breve". |

---

*Documento levantado com chamadas reais aos serviços do TSE em 28/09/2026.
DivulgaCandContas e Resultados não têm documentação oficial; as rotas foram
obtidas a partir do comportamento dos sites públicos.*
