# Open-Meteo — Forecast API

API meteorológica aberta, **gratuita e sem chave** para uso não comercial.
Usada em `agora_na_copa_2026` (`weather-core.ts`) para a condição atual nos
estádios da Copa.

- Site oficial: <https://open-meteo.com>
- Documentação: <https://open-meteo.com/en/docs>
- Licença dos dados: CC BY 4.0 (atribuição obrigatória)

---

## 1. Visão geral

Previsão e condições atuais por coordenada geográfica, sem cadastro. Cobre
temperatura, sensação térmica, umidade, vento, precipitação e o código WMO de
condição do tempo.

### URL base

```
https://api.open-meteo.com/v1/forecast
```

Todas as respostas são **JSON**. Somente `GET`.

---

## 2. Autenticação

**Nenhuma.** Não há token, cabeçalho ou cadastro para o plano gratuito.

O plano comercial usa outro host (`customer-api.open-meteo.com`) e um parâmetro
`apikey`; não é necessário para uso não comercial.

### 2.1 Configuração local

Não há credencial a proteger; `.env.example` guarda apenas a URL base e o campo
opcional de chave comercial.

```bash
cp open_meteo/.env.example open_meteo/.env
set -a && . ./open_meteo/.env && set +a
```

---

## 3. Endpoint: previsão / condição atual

### `GET /v1/forecast`

| Parâmetro         | Tipo    | Obrigatório | Descrição                                                        |
|-------------------|---------|-------------|------------------------------------------------------------------|
| `latitude`        | float   | sim         | Latitude em graus decimais                                       |
| `longitude`       | float   | sim         | Longitude em graus decimais                                      |
| `current`         | string  | não         | Lista de variáveis instantâneas, separadas por vírgula           |
| `hourly`          | string  | não         | Variáveis horárias                                               |
| `daily`           | string  | não         | Variáveis diárias (exige `timezone`)                             |
| `timezone`        | string  | não         | Fuso; `auto` resolve pelo par lat/lon                            |
| `wind_speed_unit` | string  | não         | `kmh` (padrão `ms`), `mph`, `kn`                                 |
| `temperature_unit`| string  | não         | `celsius` (padrão) ou `fahrenheit`                               |
| `forecast_days`   | int     | não         | 1–16 (padrão 7)                                                  |

### Variáveis usadas em `agora_na_copa_2026`

`temperature_2m`, `apparent_temperature`, `relative_humidity_2m`,
`weather_code`, `wind_speed_10m`, `is_day`

**Exemplo**

```bash
curl -s "$OPEN_METEO_BASE?latitude=40.8135&longitude=-74.0745\
&current=temperature_2m,apparent_temperature,relative_humidity_2m,weather_code,wind_speed_10m,is_day\
&wind_speed_unit=kmh&timezone=auto"
```

```json
{
  "latitude": 40.80899,
  "longitude": -74.06947,
  "generationtime_ms": 0.6237030029296875,
  "utc_offset_seconds": -14400,
  "timezone": "America/New_York",
  "timezone_abbreviation": "GMT-4",
  "elevation": 4.0,
  "current_units": {
    "time": "iso8601", "interval": "seconds",
    "temperature_2m": "°C", "apparent_temperature": "°C",
    "relative_humidity_2m": "%", "weather_code": "wmo code",
    "wind_speed_10m": "km/h", "is_day": ""
  },
  "current": {
    "time": "2026-09-01T09:00", "interval": 900,
    "temperature_2m": 23.9, "apparent_temperature": 28.1,
    "relative_humidity_2m": 89, "weather_code": 3,
    "wind_speed_10m": 5.5, "is_day": 1
  }
}
```

### Campos de resposta

| Campo                    | Tipo   | Descrição                                                     |
|--------------------------|--------|---------------------------------------------------------------|
| `latitude` / `longitude` | double | Coordenada da **célula de grade** usada — difere da solicitada |
| `generationtime_ms`      | double | Tempo de geração no servidor                                   |
| `utc_offset_seconds`     | int    | Offset do fuso resolvido                                       |
| `timezone`               | string | Fuso IANA resolvido (com `timezone=auto`)                      |
| `elevation`              | double | Altitude da célula, em metros                                  |
| `current_units`          | object | Unidade de cada variável em `current`                          |
| `current.time`           | string | Horário **local** da leitura (ISO 8601, sem offset)            |
| `current.interval`       | int    | Intervalo de amostragem, em segundos (900 = 15 min)            |
| `current.temperature_2m` | double | Temperatura a 2 m, em °C                                       |
| `current.apparent_temperature` | double | Sensação térmica, em °C                                  |
| `current.relative_humidity_2m` | int | Umidade relativa, em %                                      |
| `current.weather_code`   | int    | Código WMO de condição — ver seção 4                           |
| `current.wind_speed_10m` | double | Velocidade do vento a 10 m                                     |
| `current.is_day`         | int    | `1` = dia, `0` = noite                                         |

---

## 4. Códigos WMO (`weather_code`)

| Código        | Condição                                   |
|---------------|--------------------------------------------|
| `0`           | Céu limpo                                  |
| `1`, `2`, `3` | Predominantemente limpo / parcialmente nublado / nublado |
| `45`, `48`    | Névoa / névoa com geada                    |
| `51`, `53`, `55` | Garoa leve / moderada / intensa          |
| `56`, `57`    | Garoa congelante leve / intensa            |
| `61`, `63`, `65` | Chuva fraca / moderada / forte           |
| `66`, `67`    | Chuva congelante leve / forte              |
| `71`, `73`, `75` | Neve fraca / moderada / forte            |
| `77`          | Grãos de neve                              |
| `80`, `81`, `82` | Pancadas de chuva fracas / moderadas / fortes |
| `85`, `86`    | Pancadas de neve fracas / fortes           |
| `95`          | Tempestade                                 |
| `96`, `99`    | Tempestade com granizo                     |

Códigos fora dessa lista devem cair em um rótulo genérico.

---

## 5. Limites de uso

- Uso **não comercial** e abaixo de ~10.000 chamadas/dia: livre, sem chave.
- Acima disso, ou uso comercial: plano pago em <https://open-meteo.com/en/pricing>.
- Os dados são atualizados de hora em hora; `current.interval` observado = 900 s.
  Cachear por ~15 min é suficiente (é o TTL usado em `agora_na_copa_2026`).
- Atribuição CC BY 4.0 é exigida ao publicar os dados.

---

## 6. Comportamento observado (verificado em campo)

Verificado em 01/09/2026 com chamada real:

| # | Observação |
|---|------------|
| 1 | `HTTP 200` sem qualquer cabeçalho de autenticação — confirmado. |
| 2 | `latitude`/`longitude` da resposta **não** são os solicitados: a API arredonda para o centro da célula de grade (pedido `40.8135,-74.0745` → devolvido `40.80899,-74.06947`). Não use para reidentificar o ponto. |
| 3 | `current.time` vem sem offset (`"2026-09-01T09:00"`); o fuso está em `timezone` / `utc_offset_seconds`, não no timestamp. |
| 4 | `is_day` é **int** (`1`/`0`), não boolean. |
| 5 | `current_units` descreve as unidades de cada campo — útil para não presumir °C/km/h. |
| 6 | Sem o parâmetro `current`, a resposta não traz o bloco `current`; um parser deve tolerar sua ausência. |

---

*Documento compilado a partir da documentação oficial da Open-Meteo e validado
com chamadas reais. Integração de referência: `weather-core.ts` em
`agora_na_copa_2026`.*
