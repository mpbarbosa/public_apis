# MaxMind GeoLite2 — geolocalização por IP

Resolução de IP → país/cidade. Usada em `agora_na_copa_2026` para pré-selecionar
o país no guia de transmissão (`geo-core.ts`, `/api/geo`) e para as seções
"Top países" e "Top cidades" do relatório de tráfego (`scripts/traffic-report.sh`).

- Site: <https://www.maxmind.com/en/geolite2/signup>
- Documentação: <https://dev.maxmind.com/geoip/>
- Licença dos dados: GeoLite2 EULA + CC BY-SA 4.0 (atribuição obrigatória)

---

## 1. Visão geral

MaxMind oferece **três formas** de consumo, com características bem diferentes:

| Forma | Transporte | Credencial | Latência | IP do usuário sai do host? |
|-------|------------|------------|----------|----------------------------|
| **A. Banco local `.mmdb`** | Leitura de arquivo | Só para baixar | µs | **Não** |
| **B. Web service GeoLite** | HTTPS | Sim | rede | Sim |
| **C. Web service GeoIP2 Precision** | HTTPS | Sim (pago) | rede | Sim |

`agora_na_copa_2026` usa exclusivamente a **forma A**: o `.mmdb` é baixado
periodicamente e consultado em memória, de modo que **nenhum IP de visitante
trafega para a MaxMind**. As formas B e C estão documentadas aqui porque são as
APIs HTTP propriamente ditas.

### URLs base

| Finalidade                      | URL                                         |
|---------------------------------|---------------------------------------------|
| Download de bancos              | `https://download.maxmind.com/geoip/databases` |
| Web service GeoLite (gratuito)  | `https://geolite.info/geoip/v2.1`           |
| Web service GeoIP2 (pago)       | `https://geoip.maxmind.com/geoip/v2.1`      |

---

## 2. Autenticação

Todas as três formas exigem **conta MaxMind**: a forma A apenas para baixar o
banco; as formas B e C a cada requisição.

### 2.1 Como obter as credenciais

1. Crie a conta gratuita em <https://www.maxmind.com/en/geolite2/signup>
   (exige confirmação por e-mail e aceite do EULA do GeoLite2).
2. No painel, vá em **"Manage License Keys"** → **"Generate new license key"**.
3. Anote o **Account ID** (numérico) e a **License Key** — a chave só é exibida
   uma vez.

A autenticação é **HTTP Basic**, com `account_id` como usuário e `license_key`
como senha.

### 2.2 Configuração local

As credenciais ficam em `maxmind_geolite2/.env`, **fora do controle de versão**
(`.gitignore` deste diretório). Use `.env.example` como modelo:

```bash
cp maxmind_geolite2/.env.example maxmind_geolite2/.env   # preencha id/chave
chmod 600 maxmind_geolite2/.env
set -a && . ./maxmind_geolite2/.env && set +a
```

| Variável                | Descrição                                          |
|-------------------------|----------------------------------------------------|
| `MAXMIND_ACCOUNT_ID`    | Account ID numérico                                |
| `MAXMIND_LICENSE_KEY`   | License key                                        |
| `GEO_DB`                | Caminho do `.mmdb` para consulta local             |
| `GEOLITE_WS_BASE`       | `https://geolite.info/geoip/v2.1`                  |
| `MAXMIND_DOWNLOAD_BASE` | `https://download.maxmind.com/geoip/databases`     |

> O `.gitignore` também barra `*.mmdb`, `*.tar.gz` e `GeoIP.conf` — os bancos
> pesam dezenas de MB e o `GeoIP.conf` carrega a license key em texto puro.

---

## 3. Forma A — banco local `.mmdb`

### 3.1 Edições

| Edition ID          | Conteúdo                        | Tamanho aprox. |
|---------------------|----------------------------------|----------------|
| `GeoLite2-Country`  | País                             | ~9 MB          |
| `GeoLite2-City`     | País + cidade + coordenadas      | ~66 MB         |
| `GeoLite2-ASN`      | Número e nome do sistema autônomo| ~10 MB         |

`GeoLite2-City` é **superconjunto** de `GeoLite2-Country`: preferir City quando
ambos existem não perde nada e habilita a agregação por cidade — é a regra
adotada em `traffic-report.sh`.

### 3.2 Download

#### `GET /geoip/databases/{EDITION_ID}/download`

| Parâmetro | Valor      | Descrição                                   |
|-----------|------------|---------------------------------------------|
| `suffix`  | `tar.gz`   | Banco binário `.mmdb` (padrão)              |
| `suffix`  | `zip`      | Variante CSV                                |

```bash
curl -O -J -L -u "$MAXMIND_ACCOUNT_ID:$MAXMIND_LICENSE_KEY" \
  "$MAXMIND_DOWNLOAD_BASE/GeoLite2-Country/download?suffix=tar.gz"
```

`-L` é obrigatório: desde janeiro de 2024 o download redireciona para URLs
pré-assinadas em R2 (`*.r2.cloudflarestorage.com`). Libere esse host em
firewalls e proxies.

#### Checar atualização sem consumir cota

```bash
curl -I -L -u "$MAXMIND_ACCOUNT_ID:$MAXMIND_LICENSE_KEY" \
  "$MAXMIND_DOWNLOAD_BASE/GeoLite2-Country/download?suffix=tar.gz"
```

Os cabeçalhos `Last-Modified` e `Content-Disposition` (com o nome
`{Edition}_YYYYMMDD.tar.gz`) revelam a data de build.

#### `geoipupdate`

Ferramenta oficial que automatiza o ciclo. Configuração em `GeoIP.conf`:

```
AccountID   SEU_ACCOUNT_ID
LicenseKey  SUA_LICENSE_KEY
EditionIDs  GeoLite2-Country GeoLite2-City
```

Exige versão 4.x ou superior (TLS 1.2+). Versões anteriores a 2.5.0 usam as
chaves `UserId` e `ProductIds`.

### 3.3 Consulta via `mmdblookup` (CLI)

```bash
sudo apt-get install -y mmdb-bin    # fornece mmdblookup
mmdblookup --file "$GEO_DB" --ip 8.8.8.8 country names en
```

```
  "United States" <utf8_string>
```

Caminhos úteis:

| Caminho                    | Retorno                       |
|----------------------------|-------------------------------|
| `country iso_code`         | Código ISO-2                  |
| `country names en`         | Nome do país                  |
| `city names en`            | Nome da cidade (só db City)   |
| `location latitude`        | Latitude                      |
| `registered_country iso_code` | País de registro do bloco  |

Metadados do banco (tipo, data de build, idiomas):

```bash
mmdblookup --file "$GEO_DB" --ip 8.8.8.8 --verbose | head -12
```

### 3.4 Consulta via biblioteca (Node.js)

```ts
import maxmind, { type CountryResponse, type Reader } from "maxmind";

const reader: Reader<CountryResponse> = await maxmind.open<CountryResponse>(
  process.env.GEO_DB || "/var/lib/GeoIP/GeoLite2-Country.mmdb",
);
const result = reader.get("8.8.8.8");
const iso = result?.country?.iso_code ?? result?.registered_country?.iso_code ?? null;
```

O `reader` é síncrono após aberto — abra **uma vez** e reaproveite.

---

## 4. Forma B — web service GeoLite (gratuito)

### `GET /geoip/v2.1/{service}/{ip}`

Host: `https://geolite.info`

| Serviço   | Caminho                           | Conteúdo             |
|-----------|-----------------------------------|----------------------|
| `country` | `/geoip/v2.1/country/{ip}`        | País                 |
| `city`    | `/geoip/v2.1/city/{ip}`           | País + cidade        |

O IP pode ser IPv4, IPv6 (forma canônica, RFC 5952) ou a string literal **`me`**,
que resolve o IP de quem chama.

```bash
curl -s -u "$MAXMIND_ACCOUNT_ID:$MAXMIND_LICENSE_KEY" \
  "$GEOLITE_WS_BASE/country/8.8.8.8"
```

Somente HTTPS com TLS 1.2+; chamadas em HTTP recebem `403`.

---

## 5. Forma C — web service GeoIP2 Precision (pago)

Host: `https://geoip.maxmind.com`

| Serviço     | Caminho                      | Conteúdo                                  |
|-------------|------------------------------|-------------------------------------------|
| `country`   | `/geoip/v2.1/country/{ip}`   | País                                      |
| `city`      | `/geoip/v2.1/city/{ip}`      | País + cidade + coordenadas               |
| `insights`  | `/geoip/v2.1/insights/{ip}`  | Tudo acima + confiança, ISP, anonimato    |

Mesma autenticação e mesmo formato de resposta; cobra por consulta.

---

## 6. Formato da resposta (formas B e C)

```json
{
  "country":   { "iso_code": "US", "geoname_id": 6252001, "names": { "pt-BR": "Estados Unidos" } },
  "continent": { "code": "NA", "geoname_id": 6255149, "names": { "en": "North America" } },
  "city":      { "geoname_id": 5375480, "names": { "en": "Mountain View" } },
  "location":  { "latitude": 37.386, "longitude": -122.0838, "accuracy_radius": 1000, "time_zone": "America/Los_Angeles" },
  "traits":    { "ip_address": "8.8.8.8", "network": "8.8.8.0/24", "autonomous_system_number": 15169 },
  "registered_country": { "iso_code": "US" },
  "maxmind":   { "queries_remaining": 9998 }
}
```

| Objeto                | Campos principais                                                     |
|-----------------------|-----------------------------------------------------------------------|
| `country`             | `iso_code`, `geoname_id`, `names`, `confidence`                       |
| `continent`           | `code`, `geoname_id`, `names`                                         |
| `city`                | `geoname_id`, `names`, `confidence`                                   |
| `location`            | `latitude`, `longitude`, `accuracy_radius`, `time_zone`, `metro_code` |
| `traits`              | `ip_address`, `network`, `autonomous_system_number`, `user_type`      |
| `subdivisions[]`      | `iso_code`, `geoname_id`, `names`, `confidence`                       |
| `registered_country`  | País onde o provedor registrou o bloco de IP                          |
| `represented_country` | País representado (bases militares etc.)                              |
| `anonymizer`          | Detecção de VPN/proxy — só em Insights                                |
| `maxmind`             | `queries_remaining`                                                   |

`names` é um mapa por idioma (`de`, `en`, `es`, `fr`, `ja`, `pt-BR`, `ru`, `zh-CN`).

### Códigos de erro

| Código                  | HTTP | Significado                                   |
|-------------------------|------|-----------------------------------------------|
| `IP_ADDRESS_INVALID`    | 400  | IPv4/IPv6 malformado                          |
| `IP_ADDRESS_REQUIRED`   | 400  | IP não informado                              |
| `IP_ADDRESS_RESERVED`   | 400  | Faixa reservada ou privada                    |
| `SERVICE_INVALID`       | 400  | Serviço indisponível neste host               |
| `AUTHORIZATION_INVALID` | 401  | Account ID e/ou license key inválidos         |
| `LICENSE_KEY_REQUIRED`  | 401  | License key ausente no header                 |
| `ACCOUNT_ID_REQUIRED`   | 401  | Account ID ausente no header                  |
| `INSUFFICIENT_FUNDS`    | 402  | Sem créditos ou cota diária esgotada          |
| `PERMISSION_REQUIRED`   | 403  | Sem permissão para o serviço                  |
| `IP_ADDRESS_NOT_FOUND`  | 404  | IP ausente do banco                           |
| —                       | 429  | Rate limit por excesso de erros               |
| `SERVER_ERROR`          | 500  | Erro interno                                  |
| —                       | 503  | Indisponível temporariamente                  |

---

## 7. Resumo dos endpoints

| Método | URL                                                        | Credencial | Descrição             |
|--------|------------------------------------------------------------|------------|-----------------------|
| GET    | `download.maxmind.com/geoip/databases/{edition}/download`   | Basic      | Baixar banco          |
| HEAD   | idem                                                        | Basic      | Checar `Last-Modified`|
| GET    | `geolite.info/geoip/v2.1/country/{ip\|me}`                  | Basic      | País (gratuito)       |
| GET    | `geolite.info/geoip/v2.1/city/{ip\|me}`                     | Basic      | Cidade (gratuito)     |
| GET    | `geoip.maxmind.com/geoip/v2.1/country/{ip\|me}`             | Basic      | País (pago)           |
| GET    | `geoip.maxmind.com/geoip/v2.1/city/{ip\|me}`                | Basic      | Cidade (pago)         |
| GET    | `geoip.maxmind.com/geoip/v2.1/insights/{ip\|me}`            | Basic      | Insights (pago)       |
| —      | Leitura local do `.mmdb`                                    | —          | Sem rede              |

---

## 8. Limites de uso

- **Web service GeoLite:** 1.000 consultas por dia por conta, com teto de
  requisições concorrentes. Excedido → `INSUFFICIENT_FUNDS` (402).
- **Download:** cota diária por edição; use `HEAD` para checar atualização sem
  consumi-la. Excesso → `429`.
- Os bancos GeoLite2 são republicados **duas vezes por semana** (terças e
  sextas); atualizar mais de uma vez por dia é desperdício.
- A precisão do GeoLite2 é menor que a das edições pagas — adequado para
  pré-selecionar um padrão na interface, **não** para decisões de cobrança,
  conformidade ou bloqueio.
- Atribuição obrigatória (CC BY-SA 4.0) ao exibir os dados.

### Privacidade

Consultar o `.mmdb` local mantém o IP do visitante no seu host. As formas B e C
**enviam o IP do usuário final à MaxMind** — o que tem implicações de LGPD/GDPR.
Foi exatamente por isso que `agora_na_copa_2026` optou pelo banco local.

---

## 9. Comportamento observado (verificado em campo)

Verificado em 28/09/2026, com `mmdblookup` 1.12.2 e os bancos em `/var/lib/GeoIP/`:

| # | Observação |
|---|------------|
| 1 | `geolite.info/geoip/v2.1/country/8.8.8.8` sem credenciais retorna `401` com `{"code":"AUTHORIZATION_INVALID"}` — o web service "gratuito" **exige conta**, não é anônimo. |
| 2 | `download.maxmind.com/.../download` sem credenciais retorna `401` — confirmado. |
| 3 | `mmdblookup` sai com **código 5** quando o caminho não existe no registro (ex.: `city` num banco Country, ou um registro de City sem nó de cidade). Sob `set -euo pipefail` isso aborta o script — daí o `|| true` em `traffic-report.sh`. |
| 4 | IP privado (`192.168.1.1`) **não** é erro: imprime "Could not find an entry for this IP address" e sai com **código 0**. Distinga "sem entrada" de "falha". |
| 5 | O tipo do banco vem dos metadados (`Type: GeoLite2-City`), não do nome do arquivo — por isso a detecção por `mmdblookup --verbose` é mais confiável que checar o filename. |
| 6 | `--verbose` também expõe `Build epoch` (ex.: `1782800340 (2026-06-30 06:19:00 UTC)`) e os idiomas disponíveis — bom para monitorar a idade do banco. |
| 7 | O valor sai citado e tipado (`"United States" <utf8_string>`); o parse usual é `awk -F'"' 'NF>1 {print $2; exit}'`. |
| 8 | Alguns IPs só têm `registered_country`, sem `country` — encadeie o fallback, como faz `countryFromCountryResponse`. |
| 9 | IPv6 mapeado em IPv4 (`::ffff:203.0.113.7`) precisa ter o prefixo removido antes da consulta (ver `normalizeClientIp`). |
| 10 | Banco ausente deve degradar, não quebrar: em CI/Docker sem o `.mmdb`, `/api/geo` responde `country: null` e a interface cai no padrão do locale. |

---

*Documento compilado a partir da documentação oficial da MaxMind e validado com
consultas locais reais. Integração de referência: `geo-core.ts`, `/api/geo` em
`server.ts` e `scripts/traffic-report.sh` em `agora_na_copa_2026`.*
