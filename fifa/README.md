# FIFA API — `api.fifa.com/api/v3`

API de dados da FIFA: calendário, partidas ao vivo, elencos, times e o guia
"Onde Assistir". Usada em `agora_na_copa_2026` para toda a camada de dados da
Copa do Mundo de 2026.

- Interface pública: <https://www.fifa.com>

> ⚠️ **API não oficial.** É a API interna que alimenta o site da FIFA. Não há
> documentação pública, contrato ou SLA. O que segue foi levantado por
> observação de tráfego real e pode mudar sem aviso.

---

## 1. Visão geral

Duas famílias de host, com finalidades distintas:

| Finalidade                              | URL base                                     |
|-----------------------------------------|----------------------------------------------|
| Dados de partidas, times e transmissão  | `https://api.fifa.com/api/v3`                |
| Conteúdo/composição de páginas do site  | `https://cxm-api.fifa.com/fifaplusweb/api`   |

Todas as respostas são **JSON**. Somente `GET`.

### Identificadores conhecidos

| Campo           | Valor     | Significado                          |
|-----------------|-----------|--------------------------------------|
| `competitionId` | `17`      | Copa do Mundo FIFA (todas as edições)|
| `seasonId`      | `285023`  | Temporada 2026                        |
| `seasonId`      | `255711`  | Temporada 2022 (Catar)                |
| `seasonId`      | `254645`  | Temporada 2018 (Rússia)               |
| `stageId`       | `289273`  | Fase de grupos nos exemplos de 2026   |
| `country`       | `BR`      | Guia de transmissão do Brasil         |
| `language`      | `pt`      | Respostas em português                |

### Times (2026)

| Seleção  | `IdTeam` |
|----------|----------|
| Brasil   | `43924`  |
| Marrocos | `43872`  |
| França   | `43946`  |

---

## 2. Autenticação

**Nenhuma.** Não há token, chave ou cabeçalho obrigatório.

### 2.1 Configuração local

Não há credencial a proteger; `.env.example` guarda apenas as URLs base e os
identificadores do torneio.

```bash
cp fifa/.env.example fifa/.env
set -a && . ./fifa/.env && set +a
```

| Variável            | Descrição                                  |
|---------------------|--------------------------------------------|
| `FIFA_API_BASE`     | `https://api.fifa.com/api/v3`              |
| `FIFA_CXM_API_BASE` | `https://cxm-api.fifa.com/fifaplusweb/api` |
| `FIFA_COMPETITION_ID` | `17`                                     |
| `FIFA_SEASON_ID`    | `285023`                                   |

---

## 3. Calendário

### `GET /calendar/matches`

Partidas da temporada com times, placar, data, estádio e fase.

| Parâmetro       | Exemplo     | Descrição                    |
|-----------------|-------------|------------------------------|
| `language`      | `pt`        | Localização da resposta      |
| `idCompetition` | `17`        | Id da competição             |
| `idSeason`      | `285023`    | Id da temporada              |
| `idStage`       | `289273`    | Filtro de fase (opcional)    |
| `idMatch`       | `400021456` | Filtro de partida (opcional) |
| `count`         | `400`       | Máximo de itens              |

```bash
curl -s "$FIFA_API_BASE/calendar/matches?language=pt&idCompetition=17&idSeason=285023&count=400"
```

```json
{
  "ContinuationToken": "W3sidG9rZW4iOiJ+...",
  "ContinuationHash": "1902886111",
  "Results": [
    {
      "IdMatch": "400021456",
      "IdCompetition": "17",
      "IdSeason": "285023",
      "IdStage": "289273",
      "IdGroup": "289275",
      "Date": "2026-06-13T22:00:00Z",
      "StageName": [{ "Locale": "pt-BR", "Description": "Primeira fase" }],
      "Home": {
        "IdTeam": "43924",
        "Abbreviation": "BRA",
        "TeamName": [{ "Locale": "pt-BR", "Description": "Brasil" }]
      },
      "Away": {
        "IdTeam": "43872",
        "Abbreviation": "MAR",
        "TeamName": [{ "Locale": "pt-BR", "Description": "Marrocos" }]
      },
      "Stadium": {
        "Name": [{ "Locale": "pt-BR", "Description": "MetLife Stadium" }],
        "CityName": [{ "Locale": "pt-BR", "Description": "Nova York" }]
      }
    }
  ]
}
```

### `GET /calendar/{matchId}`

Metadados de calendário de uma única partida.

```bash
curl -s "$FIFA_API_BASE/calendar/400021456?language=pt"
```

---

## 4. Partida ao vivo

### `GET /live/football/{matchId}`

Estado ao vivo e eventos: escalações, gols, cartões e substituições.

| Campo                        | Descrição                                    |
|------------------------------|----------------------------------------------|
| `HomeTeam` / `AwayTeam`      | Blocos por time                              |
| `…IdTeam`                    | Id do time                                   |
| `…Abbreviation`              | Sigla de três letras                         |
| `…Players[]`                 | Escalação — **vazio até a escalação sair**   |
| `…Players[].IdPlayer`        | Id do jogador                                |
| `…Players[].ShirtNumber`     | Número da camisa                             |
| `…Players[].Position`        | Código de posição                            |
| `…Players[].PlayerName[]`    | Nome por locale                              |
| `…Players[].ShortName[]`     | Nome curto por locale                        |
| `…Players[].PlayerPicture.PictureUrl` | Foto do jogador                     |
| `…Coaches[]`                 | Comissão — já traz `PictureUrl` antes do jogo|
| `…Goals[]`                   | Gols                                         |
| `…Bookings[]`                | Cartões                                      |
| `…Substitutions[]`           | Substituições                                |

> `Players[]` só é preenchido após a divulgação oficial da escalação
> (tipicamente ~1 h antes do apito inicial). Antes disso, use o elenco
> (seção 6) para as fotos.

### `GET /timelines/{matchId}`

Linha do tempo minuto a minuto da partida.

---

## 5. Guia de transmissão ("Onde Assistir")

### `GET /watch/season/{seasonId}/{country}`

Fontes de transmissão de **todas** as partidas da temporada, por país.

```bash
curl -s "$FIFA_API_BASE/watch/season/285023/BR?language=pt"
```

```json
{
  "IdSeason": "285023",
  "IdCountryIso3166Alpha2": "BR",
  "Matches": [
    {
      "IdMatch": "400021456",
      "Date": "2026-06-13T22:00:00Z",
      "Sources": [
        {
          "IdChannel": "451",
          "Name": "Cazé TV",
          "Logo": "https://extranets.fifa.com/TvStationPhotos/451.png",
          "TvChannelUrl": "https://youtube.com/c/cazetv",
          "Url": "https://www.youtube.com/@CazeTV",
          "Language": "Portuguese Brazil"
        }
      ]
    }
  ]
}
```

### `GET /watch/match/{seasonId}/{matchId}/{country}`

Mesma informação para uma única partida.

| Campo                    | Tipo   | Descrição                        |
|--------------------------|--------|----------------------------------|
| `IdMatch`                | string | Id da partida                    |
| `IdCountryIso3166Alpha2` | string | País do guia (ISO-2)             |
| `Sources[].IdChannel`    | string | Id do canal                      |
| `Sources[].Name`         | string | Nome do canal                    |
| `Sources[].Logo`         | string | Logo em `extranets.fifa.com`     |
| `Sources[].Url`          | string | Link para assistir               |
| `Sources[].TvChannelUrl` | string | Site do canal (opcional)         |
| `Sources[].Language`     | string | Idioma da transmissão (opcional) |

---

## 6. Times e elencos

### `GET /teams/{teamId}/squad`

Elenco inscrito na competição, **com fotos**. Disponível antes da escalação —
é a melhor fonte de imagens pré-jogo.

| Parâmetro       | Exemplo  |
|-----------------|----------|
| `language`      | `pt`     |
| `idCompetition` | `17`     |
| `idSeason`      | `285023` |

```bash
curl -s "$FIFA_API_BASE/teams/43946/squad?language=pt&idCompetition=17&idSeason=285023"
```

### `GET /teams/{teamId}`

Metadados do time: nome, sigla, logo, cidade, estádio.

| Parâmetro  | Exemplo  |
|------------|----------|
| `language` | `pt`     |
| `idSeason` | `285023` |

---

## 7. Conteúdo do site (`cxm-api`)

Modelos de página usados pelo site da FIFA — composição de UI, não dados.

| Endpoint | Descrição |
|----------|-----------|
| `GET /pages/{locale}/tournaments/mens/worldcup/canadamexicousa2026` | Página do torneio |
| `GET /pages/{locale}/match-centre/match/{competitionId}/{seasonId}/{stageId}/{matchId}` | Página da partida |
| `GET /sections/matchdetails/header?locale=&competitionId=&seasonId=&stageId=&matchId=` | Cabeçalho |
| `GET /sections/matchdetails/tabs?…` | Abas |
| `GET /sections/matchdetails/videos?…` | Vídeos |

---

## 8. Resumo dos endpoints

| Método | Caminho                                       | Host        | Descrição            |
|--------|-----------------------------------------------|-------------|----------------------|
| GET    | `/calendar/matches`                           | `api`       | Lista de partidas    |
| GET    | `/calendar/{matchId}`                         | `api`       | Partida específica   |
| GET    | `/live/football/{matchId}`                    | `api`       | Dados ao vivo        |
| GET    | `/timelines/{matchId}`                        | `api`       | Linha do tempo       |
| GET    | `/watch/season/{seasonId}/{country}`          | `api`       | Guia da temporada    |
| GET    | `/watch/match/{seasonId}/{matchId}/{country}` | `api`       | Guia de uma partida  |
| GET    | `/teams/{teamId}/squad`                       | `api`       | Elenco               |
| GET    | `/teams/{teamId}`                             | `api`       | Metadados do time    |
| GET    | `/pages/{locale}/…`                           | `cxm-api`   | Conteúdo de página   |
| GET    | `/sections/matchdetails/…`                    | `cxm-api`   | Seções da página     |

---

## 9. Notas de uso

- **Campos localizados são arrays**, não strings: `TeamName`, `StageName`,
  `PlayerName` etc. seguem o formato `[{ "Locale": "pt-BR", "Description": "…" }]`.
  Selecione pelo `Locale` e tenha um fallback.
- **Ids são strings**, não inteiros (`"400021456"`, `"43924"`) — mesmo parecendo
  numéricos. Não converta.
- `Date` está em **UTC** com sufixo `Z`.
- TTLs de cache usados em `agora_na_copa_2026`: partida ao vivo 10 s, partida
  próxima (≤6 h) 30 s, partida estável 5 min, guia de transmissão 5 min.
- O projeto usa um *circuit breaker* (3 falhas → 60 s aberto), já que a API é
  volátil durante os jogos.

---

## 10. Comportamento observado (verificado em campo)

Verificado em 01/09/2026 com chamadas reais:

| # | Observação |
|---|------------|
| 1 | `GET /calendar/matches` respondeu `HTTP 200` (`application/json`, ~13,8 KB para `count=3`) sem qualquer autenticação — confirmado. |
| 2 | `cxm-api.fifa.com/fifaplusweb/api/pages/pt/tournaments/…` também respondeu `HTTP 200` sem autenticação. |
| 3 | A resposta de `/calendar/matches` traz `ContinuationToken` e `ContinuationHash` **antes** de `Results` — a lista é paginada; `count` alto evita a paginação, mas o token deve ser tratado se você a usar. |
| 4 | Cada resultado traz um bloco `Weather` embutido — a API já devolve condição do tempo por partida. |
| 5 | A API pode ser inalcançável a partir de certas redes (dev/preview). `agora_na_copa_2026` mantém um proxy de passagem (`/api/fifa-proxy`) num host que alcança a FIFA, como fallback. |
| 6 | Fotos de jogador e técnico vêm de `digitalhub.fifa.com`; logos de canal, de `extranets.fifa.com`. |

---

*Documento compilado a partir de `FIFA_API_DOCUMENTATION.md` de
`agora_na_copa_2026` e validado com chamadas reais. Não há fonte oficial a
citar — a API não é documentada publicamente pela FIFA.*
