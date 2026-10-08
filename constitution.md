---
title: "Constitution — Regras Persistentes do Projeto (Zona Azul Digital)"
type: knowledge
status: done
area: resources
resource: talks
tags:
  - kind/knowledge
  - area/resources
  - resource/talks
  - status/done
created: 2026-10-07
updated: 2026-10-07
---
# Constitution — Regras persistentes do projeto

Estas regras valem para todo arquivo gerado e para qualquer decisão não coberta explicitamente em `spec.md`, `plan.md` ou `tests.md`.

## Parâmetros da variante (valores fixos deste projeto)

| Parâmetro | Valor | Observação |
| --- | --- | --- |
| `TARIFA_HORA_CENTAVOS` | **400** | hora cheia |
| `FRACAO_MINUTOS` | **30** | granularidade de cobrança |
| `TETO_DIARIO_CENTAVOS` | **5000** | máximo por bilhete |
| `TOLERANCIA_MINUTOS` | **10** | minutos grátis por bilhete |
| `PORTA_SERVICO` | **8001** | porta do serviço |
| valor da fração (derivado) | **200** | `400 ÷ (60 ÷ 30)` |

