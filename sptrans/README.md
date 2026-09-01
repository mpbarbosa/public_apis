# API Olho Vivo — SPTrans

Documentação de referência da API pública de transporte coletivo da SPTrans
(São Paulo Transporte S.A.), conhecida como **API do Olho Vivo**.

- Fonte oficial: <https://www.sptrans.com.br/desenvolvedores/api-do-olho-vivo-guia-de-referencia/documentacao-api/>
- Portal do desenvolvedor: <https://www.sptrans.com.br/desenvolvedores/>

---

## 1. Visão geral

A API do Olho Vivo expõe, em tempo real, dados de:

- linhas de ônibus e seus sentidos;
- pontos de parada;
- corredores;
- empresas operadoras;
- posição GPS dos veículos;
- previsão de chegada nos pontos;
- velocidade média nas vias (arquivos KMZ).

### URL base

```
https://api.olhovivo.sptrans.com.br/v2.1
```

A versão `v0` também existe, mas `v2.1` é a versão corrente.

> **HTTPS obrigatório.** A API adotou o protocolo HTTPS e o acesso via HTTP
> simples foi desativado a partir de 02/01/2024.

### Formato

Todas as respostas (exceto os endpoints KMZ) são **JSON**.
Os nomes dos campos são abreviados — ver a tabela de cada recurso.

---

## 2. Autenticação

O acesso exige um **token** gerado na área "Meus Aplicativos" do portal do
desenvolvedor da SPTrans. A autenticação é feita uma vez por sessão; as
requisições seguintes reutilizam o cookie de sessão retornado pelo servidor.

### 2.1 Como obter o token

O token é emitido pelo **DevPlace da SPTrans**, gratuitamente:

1. **Crie uma conta** em <https://www.sptrans.com.br/desenvolvedores/cadastro-desenvolvedores/>
   e confirme o cadastro por e-mail. Já tem conta? Faça login em
   <https://www.sptrans.com.br/desenvolvedores/login-desenvolvedores/>.
2. **Acesse a área "Meus Aplicativos"**, disponível após o cadastro validado.
3. **Cadastre um aplicativo.** Segundo o guia oficial: *"Cada aplicativo
   cadastrado receberá uma chave de acesso que deverá ser utilizada para
   efetuar a autenticação."*

Observações:

- A chave é **por aplicativo**, não por conta — cadastre aplicativos distintos
  se precisar de chaves separadas (ex.: dev e produção).
- Recuperação de senha: <https://www.sptrans.com.br/desenvolvedores/esqueci-minha-senha>
- O guia de referência não publica limites de requisição nem custos.
- Trate o token como segredo: mantenha-o em variável de ambiente, fora do
  controle de versão.

### `POST /Login/Autenticar`

| Parâmetro | Tipo   | Local | Descrição                                          |
|-----------|--------|-------|----------------------------------------------------|
| `token`   | string | query | Chave de acesso gerada em "Meus Aplicativos"       |

**Requisição**

```
POST https://api.olhovivo.sptrans.com.br/v2.1/Login/Autenticar?token=SEU_TOKEN
```

**Resposta:** `true` (autenticado) ou `false` (falha).

O cookie de sessão devolvido (`apiCredentials`, `HttpOnly`, `SameSite=Lax`,
`path=/`) deve ser enviado em todas as chamadas seguintes; sem ele a API
responde `401 Unauthorized`.

### 2.2 Configuração local

O token fica em `sptrans/.env`, **fora do controle de versão**
(`.gitignore` deste diretório). Use `.env.example` como modelo:

```bash
cp sptrans/.env.example sptrans/.env   # e preencha SPTRANS_TOKEN
chmod 600 sptrans/.env
```

| Variável           | Descrição                                      |
|--------------------|------------------------------------------------|
| `SPTRANS_TOKEN`    | Chave de acesso do aplicativo                  |
| `SPTRANS_API_BASE` | URL base da API (`.../v2.1`)                   |

Carregue no shell antes de chamar a API:

```bash
set -a && . ./sptrans/.env && set +a
```

**Exemplo com cURL**

```bash
curl -s -c cookies.txt -X POST -H "Content-Length: 0" \
  "$SPTRANS_API_BASE/Login/Autenticar?token=$SPTRANS_TOKEN"
```

> ⚠️ O header `Content-Length: 0` é **obrigatório**. Sem ele o servidor
> responde `411 Length Required` (o POST não tem corpo). Em cURL, `-d ''`
> também resolve.

```bash
curl -s -b cookies.txt "$SPTRANS_API_BASE/Linha/Buscar?termosBusca=8000"
```

---

## 3. Linhas

### `GET /Linha/Buscar`

Busca linhas pelo número ou pelo nome (aceita busca parcial).

| Parâmetro     | Tipo   | Descrição                                    |
|---------------|--------|----------------------------------------------|
| `termosBusca` | string | Denominação, letreiro ou parte do nome/número |

### `GET /Linha/BuscarLinhaSentido`

Mesma busca, restrita a um sentido de operação.

| Parâmetro     | Tipo   | Descrição                                                     |
|---------------|--------|---------------------------------------------------------------|
| `termosBusca` | string | Denominação, letreiro ou parte do nome/número                  |
| `sentido`     | byte   | `1` = Principal → Secundário; `2` = Secundário → Principal      |

### Objeto de resposta (linha)

| Campo | Tipo   | Descrição                                                                                  |
|-------|--------|--------------------------------------------------------------------------------------------|
| `cl`  | int    | Código identificador da linha — **único por sentido** de operação                            |
| `lc`  | bool   | Indica se a linha opera em modo circular (sem terminal secundário)                          |
| `lt`  | string | Primeira parte do letreiro numérico da linha                                                |
| `tl`  | int    | Segunda parte do letreiro numérico: `10` = BASE, `21`/`23`/`32`/`41` = ATENDIMENTO           |
| `sl`  | int    | Sentido de operação: `1` = Principal → Secundário, `2` = Secundário → Principal              |
| `tp`  | string | Letreiro descritivo no sentido Principal → Secundário                                       |
| `ts`  | string | Letreiro descritivo no sentido Secundário → Principal                                       |

**Exemplo**

```bash
curl -s -b cookies.txt "$SPTRANS_API_BASE/Linha/Buscar?termosBusca=8000"
```

```json
[
  {
    "cl": 1273,
    "lc": false,
    "lt": "8000",
    "tl": 10,
    "sl": 1,
    "tp": "PCA.RAMOS DE AZEVEDO",
    "ts": "TERM. LAPA"
  }
]
```

---

## 4. Paradas

### `GET /Parada/Buscar`

| Parâmetro     | Tipo   | Descrição                             |
|---------------|--------|---------------------------------------|
| `termosBusca` | string | Nome da parada ou endereço de correspondência |

### `GET /Parada/BuscarParadasPorLinha`

| Parâmetro      | Tipo | Descrição                    |
|----------------|------|------------------------------|
| `codigoLinha`  | int  | Código da linha (campo `cl`) |

### `GET /Parada/BuscarParadasPorCorredor`

| Parâmetro         | Tipo | Descrição                        |
|-------------------|------|----------------------------------|
| `codigoCorredor`  | int  | Código do corredor (campo `cc`)  |

### Objeto de resposta (parada)

| Campo | Tipo   | Descrição                          |
|-------|--------|------------------------------------|
| `cp`  | int    | Código identificador da parada     |
| `np`  | string | Nome da parada                     |
| `ed`  | string | Endereço de localização da parada  |
| `py`  | double | Latitude                           |
| `px`  | double | Longitude                          |

**Exemplo**

```json
[
  {
    "cp": 340015329,
    "np": "AFONSO BRAZ B/C1",
    "ed": "R ARMINDA/ R BALTHAZAR DA VEIGA",
    "py": -23.592938,
    "px": -46.681944
  }
]
```

---

## 5. Corredores

### `GET /Corredor`

Sem parâmetros. Retorna a relação de corredores inteligentes.

| Campo | Tipo   | Descrição                        |
|-------|--------|----------------------------------|
| `cc`  | int    | Código identificador do corredor |
| `nc`  | string | Nome do corredor                 |

---

## 6. Empresas

### `GET /Empresa`

Sem parâmetros. Retorna as empresas operadoras agrupadas por área de operação.

| Campo | Tipo     | Descrição                                    |
|-------|----------|----------------------------------------------|
| `hr`  | string   | Horário de referência da consulta            |
| `e`   | object[] | Lista de áreas de operação                   |
| `e[].a` | int    | Código da área de operação                   |
| `e[].e` | object[] | Empresas da área                           |
| `e[].e[].a` | int | Código da área de operação                  |
| `e[].e[].c` | int | Código de referência da empresa             |
| `e[].e[].n` | string | Nome da empresa                          |

---

## 7. Posição dos veículos

### `GET /Posicao`

Retorna a posição de **todos** os veículos monitorados.

### `GET /Posicao/Linha`

| Parâmetro     | Tipo | Descrição                    |
|---------------|------|------------------------------|
| `codigoLinha` | int  | Código da linha (campo `cl`) |

Resposta: `hr` + array `vs` (sem o agrupamento por linha).

### `GET /Posicao/Garagem`

Veículos localizados dentro da garagem de uma empresa.

| Parâmetro       | Tipo | Obrigatório | Descrição                          |
|-----------------|------|-------------|------------------------------------|
| `codigoEmpresa` | int  | sim         | Código da empresa (campo `c`)      |
| `codigoLinha`   | int  | não         | Filtra por linha (campo `cl`)      |

### Objeto de resposta (posição)

| Campo        | Tipo     | Descrição                                                        |
|--------------|----------|------------------------------------------------------------------|
| `hr`         | string   | Horário de referência da geração das informações                  |
| `l`          | object[] | Lista de linhas localizadas                                       |
| `l[].c`      | string   | Letreiro completo                                                 |
| `l[].cl`     | int      | Código identificador da linha                                     |
| `l[].sl`     | int      | Sentido de operação (`1` ou `2`)                                  |
| `l[].lt0`    | string   | Letreiro de destino da linha                                      |
| `l[].lt1`    | string   | Letreiro de origem da linha                                       |
| `l[].qv`     | int      | Quantidade de veículos localizados                                |
| `l[].vs`     | object[] | Lista de veículos localizados                                     |
| `l[].vs[].p` | int      | Prefixo do veículo                                                |
| `l[].vs[].a` | bool     | Indica se o veículo é acessível para pessoas com deficiência      |
| `l[].vs[].ta`| string   | Horário universal (UTC) da captura da localização — ISO 8601      |
| `l[].vs[].py`| double   | Latitude do veículo                                               |
| `l[].vs[].px`| double   | Longitude do veículo                                              |

**Exemplo**

```json
{
  "hr": "11:30",
  "l": [
    {
      "c": "5010-10",
      "cl": 33887,
      "sl": 1,
      "lt0": "METRO JABAQUARA",
      "lt1": "TERM. STO. AMARO",
      "qv": 2,
      "vs": [
        {
          "p": 68021,
          "a": true,
          "ta": "2017-05-12T14:30:37Z",
          "py": -23.678712500000003,
          "px": -46.65674
        }
      ]
    }
  ]
}
```

---

## 8. Previsão de chegada

### `GET /Previsao`

Previsão de chegada de uma linha específica em uma parada específica.

| Parâmetro      | Tipo | Descrição                     |
|----------------|------|-------------------------------|
| `codigoParada` | int  | Código da parada (campo `cp`) |
| `codigoLinha`  | int  | Código da linha (campo `cl`)  |

### `GET /Previsao/Linha`

Previsão de chegada de uma linha em **todos** os pontos do seu trajeto.

| Parâmetro     | Tipo | Descrição                    |
|---------------|------|------------------------------|
| `codigoLinha` | int  | Código da linha (campo `cl`) |

Resposta: `hr` + array `ps` de pontos (`cp`, `np`, `py`, `px`, `vs`).

### `GET /Previsao/Parada`

Previsão de chegada de **todas** as linhas que atendem uma parada.

| Parâmetro      | Tipo | Descrição                     |
|----------------|------|-------------------------------|
| `codigoParada` | int  | Código da parada (campo `cp`) |

### Objeto de resposta (previsão)

| Campo           | Tipo     | Descrição                                                   |
|-----------------|----------|-------------------------------------------------------------|
| `hr`            | string   | Horário de referência da geração das informações             |
| `p`             | object   | Ponto de parada pesquisado                                   |
| `p.cp`          | int      | Código identificador da parada                               |
| `p.np`          | string   | Nome da parada                                               |
| `p.py`          | double   | Latitude da parada                                           |
| `p.px`          | double   | Longitude da parada                                          |
| `p.l`           | object[] | Lista de linhas localizadas                                  |
| `p.l[].c`       | string   | Letreiro completo                                            |
| `p.l[].cl`      | int      | Código identificador da linha                                |
| `p.l[].sl`      | int      | Sentido de operação (`1` ou `2`)                             |
| `p.l[].lt0`     | string   | Letreiro de destino da linha                                 |
| `p.l[].lt1`     | string   | Letreiro de origem da linha                                  |
| `p.l[].qv`      | int      | Quantidade de veículos localizados                           |
| `p.l[].vs`      | object[] | Lista de veículos localizados                                |
| `p.l[].vs[].p`  | int      | Prefixo do veículo                                           |
| `p.l[].vs[].t`  | string   | **Horário previsto** para a chegada do veículo na parada     |
| `p.l[].vs[].a`  | bool     | Indica se o veículo é acessível para pessoas com deficiência |
| `p.l[].vs[].ta` | string   | Horário universal (UTC) da captura — ISO 8601                |
| `p.l[].vs[].py` | double   | Latitude do veículo                                          |
| `p.l[].vs[].px` | double   | Longitude do veículo                                         |

**Exemplo**

```json
{
  "hr": "11:30",
  "p": {
    "cp": 4200953,
    "np": "PARADA ROBERTO SELMI DEI B/C",
    "py": -23.675901,
    "px": -46.752812,
    "l": [
      {
        "c": "7021-10",
        "cl": 1989,
        "sl": 1,
        "lt0": "TERM. JD. ANGELA",
        "lt1": "TERM. STO. AMARO",
        "qv": 1,
        "vs": [
          {
            "p": "74558",
            "t": "23:11",
            "a": false,
            "ta": "2017-05-07T02:11:04Z",
            "py": -23.676876,
            "px": -46.7509885
          }
        ]
      }
    ]
  }
}
```

---

## 9. Velocidade nas vias (KMZ)

Retornam arquivos **KMZ** (não JSON) com a velocidade média das vias.

| Endpoint                | Descrição                                        |
|-------------------------|--------------------------------------------------|
| `GET /KMZ`              | Todas as vias                                    |
| `GET /KMZ/Corredor`     | Somente corredores                               |
| `GET /KMZ/OutrasVias`   | Vias que não são corredores                      |

Cada um aceita o sentido como segmento de caminho opcional:

| Sentido | Caminho                | Descrição                    |
|---------|------------------------|------------------------------|
| `BC`    | `/KMZ/BC`              | Bairro → Centro              |
| `CB`    | `/KMZ/CB`              | Centro → Bairro              |

Exemplos: `/KMZ/Corredor/BC`, `/KMZ/OutrasVias/CB`.

---

## 10. Resumo dos endpoints

| Método | Caminho                                | Recurso              |
|--------|----------------------------------------|----------------------|
| POST   | `/Login/Autenticar`                    | Autenticação         |
| GET    | `/Linha/Buscar`                        | Linhas               |
| GET    | `/Linha/BuscarLinhaSentido`            | Linhas               |
| GET    | `/Parada/Buscar`                       | Paradas              |
| GET    | `/Parada/BuscarParadasPorLinha`        | Paradas              |
| GET    | `/Parada/BuscarParadasPorCorredor`     | Paradas              |
| GET    | `/Corredor`                            | Corredores           |
| GET    | `/Empresa`                             | Empresas             |
| GET    | `/Posicao`                             | Posição de veículos  |
| GET    | `/Posicao/Linha`                       | Posição de veículos  |
| GET    | `/Posicao/Garagem`                     | Posição de veículos  |
| GET    | `/Previsao`                            | Previsão de chegada  |
| GET    | `/Previsao/Linha`                      | Previsão de chegada  |
| GET    | `/Previsao/Parada`                     | Previsão de chegada  |
| GET    | `/KMZ`, `/KMZ/Corredor`, `/KMZ/OutrasVias` | Velocidade nas vias |

---

## 11. Notas de uso

- O campo `cl` (código da linha) é **único por sentido**: a mesma linha ida e
  volta possui dois códigos distintos.
- O fluxo típico é: autenticar → `Linha/Buscar` para obter `cl` →
  `Parada/BuscarParadasPorLinha` para obter `cp` → `Previsao` ou `Posicao`.
- Os horários em `ta` estão em UTC (ISO 8601); `hr` e `t` estão no horário
  local de São Paulo (`HH:mm`).
- A sessão expira; ao receber `401`, reautentique e repita a chamada.

---

## 12. Comportamento observado (verificado em campo)

Diferenças entre o guia oficial e o comportamento real da API, confirmadas
chamando os 14 endpoints em 01/09/2026:

| # | Observação |
|---|------------|
| 1 | `POST /Login/Autenticar` exige `Content-Length: 0`; sem ele retorna `411 Length Required`. |
| 2 | O cookie de sessão chama-se **`apiCredentials`** (`HttpOnly`, `SameSite=Lax`, `path=/`). |
| 3 | Chamadas sem cookie retornam `401` — confirmado em `/Corredor`. |
| 4 | Os objetos de veículo trazem dois campos **não documentados**: `sv` e `is` (ambos `null` nas amostras coletadas). |
| 5 | O prefixo do veículo (`p`) vem como **string** em `/Posicao/Linha` e `/Previsao/*` (`"11525"`), mas como **int** em `/Posicao/Garagem` (`82500`). Faça o parse de forma tolerante. |
| 6 | O campo `tl` assume valores fora da lista documentada — foi observado `tl: 1` além de `10`. Trate como inteiro livre, não como enum fechado. |
| 7 | `hr` está no horário local de São Paulo e `ta` em UTC: observado `hr: "09:58"` com `ta: "2026-09-01T12:58:35Z"`. |
| 8 | `/KMZ/Corredor/BC` devolve `Content-Type: application/vnd.google-earth.kmz` (~20 KB) — trate como binário, não como JSON. |
| 9 | A API está atrás da Cloudflare; erros de infraestrutura vêm em HTML, não em JSON. |

### Amostra real

```bash
curl -s -b cookies.txt \
  "https://api.olhovivo.sptrans.com.br/v2.1/Posicao/Linha?codigoLinha=1273"
```

```json
{
  "hr": "09:58",
  "vs": [
    {
      "p": "11525",
      "a": true,
      "ta": "2026-09-01T12:58:35Z",
      "py": -23.530067375,
      "px": -46.66720975,
      "sv": null,
      "is": null
    }
  ]
}
```

---

*Documento compilado a partir do guia de referência oficial da SPTrans e
validado com chamadas reais à API. Em caso de divergência na especificação,
a fonte oficial prevalece; a seção 12 registra o comportamento observado.*
