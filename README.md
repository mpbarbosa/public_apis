# public_apis

Documentação de referência de APIs públicas, uma pasta por API.

## Índice

| API | Pasta | Autenticação | Chave necessária |
|-----|-------|--------------|------------------|
| [SPTrans — Olho Vivo](sptrans/README.md) | `sptrans/` | Token → cookie de sessão | Sim |
| [FIFA](fifa/README.md) | `fifa/` | Nenhuma | Não |
| [Open-Meteo](open_meteo/README.md) | `open_meteo/` | Nenhuma | Não |
| [Reddit](reddit/README.md) | `reddit/` | OAuth2 app-only | Sim |
| [Google Trends](google_trends/README.md) | `google_trends/` | Nenhuma | Não |
| [Wikipedia + Wikidata](wikipedia_wikidata/README.md) | `wikipedia_wikidata/` | Nenhuma (User-Agent obrigatório) | Não |
| [MaxMind GeoLite2](maxmind_geolite2/README.md) | `maxmind_geolite2/` | HTTP Basic (account ID + license key) | Sim |
| [TSE — Tribunal Superior Eleitoral](tse/README.md) | `tse/` | Nenhuma (bloqueio anti-bot Akamai) | Não |

## Padrão de cada pasta

| Arquivo        | Papel                                                        |
|----------------|--------------------------------------------------------------|
| `README.md`    | Documentação: visão geral, autenticação, endpoints, campos, limites e comportamento observado |
| `.env.example` | Modelo de configuração, versionado                            |
| `.env`         | Configuração real com credenciais — **ignorado pelo git**      |
| `.gitignore`   | Garante que `.env` nunca seja versionado                       |

Toda documentação traz uma seção final **"Comportamento observado"** com as
divergências entre a documentação oficial e o comportamento real da API,
levantadas com chamadas reais.

### Uso

```bash
cp <api>/.env.example <api>/.env    # preencha as credenciais, se houver
set -a && . ./<api>/.env && set +a
```

## Origem

`sptrans/` foi documentada a partir do guia de referência oficial da SPTrans.
As demais foram levantadas a partir das integrações do projeto
[`agora_na_copa_2026`](https://github.com/mpbarbosa/agora_na_copa_2026) e
validadas com chamadas reais.

`maxmind_geolite2/` cobre as três formas de consumo — banco `.mmdb` local
(a usada naquele projeto, sem rede), API de download e web services HTTP.
