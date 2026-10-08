# Spec — API de bilhetes de estacionamento rotativo (Zona Azul Digital)

API REST em JSON. Base URL `http://localhost:8001`. Valores da variante estão na `constitution.md` e valem aqui: tarifa **400**, fração **30 min**, teto **5000**, tolerância **10 min**, porta **8001**.

## Modelo de dados

Bilhete: `id` (inteiro sequencial a partir de 1, nunca reutilizado), `placa` (string), `entrada` (datetime com `-03:00`), `status` (`aberto` | `encerrado` | `cancelado`), `saida` e `minutos` e `valor_centavos` (só após encerrar).

| Status | Chaves da representação do bilhete (em listagens e UC1/UC5) |
| --- | --- |
| `aberto` | `id`, `placa`, `entrada`, `status` |
| `cancelado` | `id`, `placa`, `entrada`, `status` (sem `saida`, `minutos` e `valor_centavos`) |
| `encerrado` | `id`, `placa`, `entrada`, `status`, `saida`, `minutos`, `valor_centavos` |

> [!NOTE]
> Exceção: a resposta do UC2 tem exatamente as seis chaves do contrato (`id`, `placa`, `entrada`, `saida`, `minutos`, `valor_centavos`), sem `status`.

## Regra de cobrança (usada no UC2 e no UC4)

1. `minutos` = parte inteira (truncada) de `(saida − entrada)` em minutos; nunca negativo (`entrada` no futuro → `minutos = 0`). Segundos sobrando são ignorados.
2. Se `minutos ≤ 10` → `valor_centavos = 0` (tolerância).
3. Senão: `fracoes = ceil(minutos / 30)` calculado com inteiros `(minutos + 29) // 30`; `valor = fracoes × 200` (200 = valor da fração = `400 // 2`).
4. `valor_centavos = min(valor, 5000)`.

> [!WARNING]
> Teto: `valor_centavos` jamais passa de 5000, por maior que seja a duração (o teto é por bilhete, não multiplica por dia). Atingido exatamente com 25 frações (750 min); 751 min continua 5000.

> [!WARNING]
> Tolerância não é desconto: com 11 minutos cobra-se a fração inteira (200), nunca "11 − 10". Com 10 minutos exatos o valor é 0.

Tabela de referência (obrigatória, deve bater exatamente):

| `minutos` | frações | `valor_centavos` |
| --- | --- | --- |
| 0 | — | 0 |
| 10 | — | 0 |
| 11 | 1 | 200 |
| 30 | 1 | 200 |
| 31 | 2 | 400 |
| 60 | 2 | 400 |
| 61 | 3 | 600 |
| 95 | 4 | 800 |
| 750 | 25 | 5000 |
| 751 | 26 | 5000 (teto) |
| 5000 | 167 | 5000 (teto) |

## UC1 — Abrir bilhete

`POST /bilhetes` — body JSON `{"placa": "ABC1D23"}`, com `entrada` opcional.

- Placa válida: string de exatamente 7 caracteres que casam `^[A-Z0-9]{7}$` (letras maiúsculas e dígitos). Minúsculas, espaços, hífen, tamanho ≠ 7, número JSON, `null` ou ausência → inválida. A placa não é normalizada.
- `entrada` opcional: ausente ou `null` → instante atual (truncado em segundos, fuso `-03:00`). Presente → deve ser string ISO-8601 com horário (regex `^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}(:\d{2}(\.\d+)?)?(Z|[+-]\d{2}:\d{2})?$` e data/hora reais). Sem fuso → assume `-03:00`. Com outro fuso → converte para `-03:00`. Pode estar no passado (é o gancho de teste). Valor não-string, vazio, só data (`2026-10-05`) ou texto livre → inválida.
- Body ausente, não-JSON ou que não seja objeto → `placa_invalida`.

Critérios de aceite:

| # | Dado / Quando | Então |
| --- | --- | --- |
| UC1.1 | `{"placa":"ABC1D23"}` e placa sem bilhete aberto | `201` + `{"id":1,"placa":"ABC1D23","entrada":"<agora -03:00>","status":"aberto"}` |
| UC1.2 | `entrada` = `2026-10-12T08:30:00-03:00` | `201` e `entrada` retornada = `2026-10-12T08:30:00-03:00` |
| UC1.3 | `entrada` = `2026-10-12T11:30:00Z` | `201` e `entrada` retornada = `2026-10-12T08:30:00-03:00` |
| UC1.4 | placa ausente, minúscula, com 6 ou 8 caracteres, com símbolo | `422` `{"erro":"placa_invalida"}` |
| UC1.5 | `entrada` = `"ontem"` ou `"2026-13-40T00:00:00-03:00"` | `422` `{"erro":"entrada_invalida"}` |
| UC1.6 | placa inválida **e** entrada inválida juntas | `422` `placa_invalida` (placa é validada primeiro) |
| UC1.7 | ids em sequência | 1º bilhete `id=1`, 2º `id=2`, … |

## UC2 — Encerrar bilhete

`POST /bilhetes/{id}/encerramento` (sem body). `saida` = instante atual (`-03:00`, segundos). Aplica a Regra de cobrança. O bilhete passa a `encerrado` e a placa fica livre.

Resposta `200`: `{"id":1,"placa":"ABC1D23","entrada":"...","saida":"...","minutos":95,"valor_centavos":800}`.

| # | Dado / Quando | Então |
| --- | --- | --- |
| UC2.1 | bilhete aberto há 95 min | `200`, `minutos=95`, `valor_centavos=800` |
| UC2.2 | bilhete aberto há 30 min | `200`, `valor_centavos=200` (fração exata = 1 fração) |
| UC2.3 | bilhete aberto há 31 min | `200`, `valor_centavos=400` (+1 min cobra a seguinte) |
| UC2.4 | bilhete aberto há 3 dias | `200`, `valor_centavos=5000` (teto) |
| UC2.5 | `valor_centavos` na resposta | sempre `int` JSON (sem `.`), e `saida`, `entrada` com `-03:00` |
| UC2.6 | `id` inexistente (ou não numérico, ex.: `abc`) | `404` `{"erro":"bilhete_nao_encontrado"}` |
| UC2.7 | bilhete já encerrado ou cancelado | `409` `{"erro":"bilhete_ja_encerrado"}` |

## UC3 — Listar ativos

`GET /bilhetes/ativos` → `200` com array de bilhetes `aberto`, **mais recentes primeiro** (ordenar por `entrada` decrescente; empate por `id` decrescente). Sem ativos → `[]`. Encerrados e cancelados nunca aparecem. A rota `/bilhetes/ativos` tem precedência sobre qualquer rota com parâmetro de caminho.

| # | Dado / Quando | Então |
| --- | --- | --- |
| UC3.1 | 2 abertos, 1 encerrado, 1 cancelado | `200` com exatamente os 2 abertos |
| UC3.2 | abertos com entradas 10:00 e 11:00 | o de 11:00 vem primeiro |
| UC3.3 | nenhum aberto | `200` `[]` |

## UC4 — Relatório diário

`GET /relatorios/diario?data=AAAA-MM-DD` → `200` `{"data","total_bilhetes","faturamento_centavos","tempo_medio_minutos"}`.

- "Dia do bilhete" = **data da `entrada` no fuso `-03:00`** (a `saida` não define o dia).
- `data`: regex `^\d{4}-\d{2}-\d{2}$` e data de calendário real (`2026-02-30` é inválida). Ausente → inválida.
- `total_bilhetes`: quantidade de bilhetes do dia, qualquer status (abertos, encerrados e cancelados).
- `faturamento_centavos`: soma de `valor_centavos` dos bilhetes encerrados do dia (inteiro).
- `tempo_medio_minutos`: média de `minutos` dos bilhetes encerrados do dia, inteiro, 0,5 arredonda para cima; sem encerrados → `0`. Cálculo inteiro: `(2*soma + n) // (2*n)`.
- Dia sem bilhetes → `200` com `total_bilhetes=0`, `faturamento_centavos=0`, `tempo_medio_minutos=0`.

| # | Dado / Quando | Então |
| --- | --- | --- |
| UC4.1 | encerrados com 10 e 11 min no dia | `tempo_medio_minutos=11` (10,5 → 11) |
| UC4.2 | encerrados com 10, 10 e 11 min | `tempo_medio_minutos=10` (10,33 → 10) |
| UC4.3 | 1 encerrado (valor 400), 1 aberto, 1 cancelado, mesmo dia | `total_bilhetes=3`, `faturamento_centavos=400` |
| UC4.4 | bilhetes de outro dia | não entram em nenhum campo |
| UC4.5 | `data=2026-13-01`, `data=05/10/2026` ou sem `data` | `422` `{"erro":"data_invalida"}` |
| UC4.6 | dia sem bilhetes | `200` com zeros e `data` ecoada |

## UC5 — Cancelar bilhete

`POST /bilhetes/{id}/cancelamento` (sem body). Só bilhete **aberto**. Sem cobrança: não gera `saida`, `minutos` nem `valor_centavos`. A placa fica livre.

| # | Dado / Quando | Então |
| --- | --- | --- |
| UC5.1 | bilhete aberto | `200` `{"id","placa","entrada","status":"cancelado"}` |
| UC5.2 | `id` inexistente | `404` `{"erro":"bilhete_nao_encontrado"}` |
| UC5.3 | bilhete encerrado ou já cancelado | `409` `{"erro":"bilhete_nao_aberto"}` |
| UC5.4 | bilhete cancelado | não aparece em UC3 e não conta no faturamento do UC4 |

## UC6 — Histórico por placa

`GET /bilhetes?placa=ABC1D23` → `200` com array de todos os bilhetes da placa (qualquer status), mais recentes primeiro (mesma ordenação do UC3). Placa válida sem histórico → `[]`. Parâmetro `placa` ausente ou fora do formato da placa (mesma regra do UC1) → `422` `placa_invalida`.

| # | Dado / Quando | Então |
| --- | --- | --- |
| UC6.1 | placa com 1 encerrado, 1 cancelado, 1 aberto | `200` com os 3, recente primeiro, cada um na representação do seu status |
| UC6.2 | placa nunca usada (válida) | `200` `[]` |
| UC6.3 | `?placa=abc` ou sem `placa` | `422` `placa_invalida` |
| UC6.4 | bilhetes de outras placas | não aparecem |

## UC7 — Tolerância gratuita

Implementada dentro da Regra de cobrança (passo 2). Duração `≤ 10` min → `valor_centavos = 0` e o bilhete é `encerrado` normalmente (`minutos` real é retornado).

| # | `minutos` | Esperado |
| --- | --- | --- |
| UC7.1 | 0 | `0` |
| UC7.2 | 10 | `0` |
| UC7.3 | 11 | `200` (integral, sem desconto) |

## UC8 — Uma vaga por placa

`POST /bilhetes` para placa que já tem bilhete aberto → `409` `{"erro":"bilhete_em_aberto"}`. A checagem ocorre depois das validações 422. Após encerrar ou cancelar, a mesma placa pode abrir de novo (novo `id`). Placas diferentes não interferem.

| # | Dado / Quando | Então |
| --- | --- | --- |
| UC8.1 | placa com bilhete aberto, novo `POST` | `409` `bilhete_em_aberto` |
| UC8.2 | após encerrar, novo `POST` | `201` com novo `id` |
| UC8.3 | após cancelar, novo `POST` | `201` com novo `id` |
| UC8.4 | placa A aberta, `POST` da placa B | `201` |
| UC8.5 | placa aberta + `entrada` inválida | `422` `entrada_invalida` (422 antes de 409) |

## Tabela geral de erros

| Situação | Status | Body |
| --- | --- | --- |
| Placa ausente ou inválida (POST `/bilhetes`, GET `/bilhetes`) | 422 | `{"erro":"placa_invalida"}` |
| `entrada` fora de ISO-8601 | 422 | `{"erro":"entrada_invalida"}` |
| `data` fora de AAAA-MM-DD ou inexistente no calendário | 422 | `{"erro":"data_invalida"}` |
| Bilhete inexistente (encerrar, cancelar) | 404 | `{"erro":"bilhete_nao_encontrado"}` |
| Encerrar bilhete já encerrado ou cancelado | 409 | `{"erro":"bilhete_ja_encerrado"}` |
| Cancelar bilhete não aberto | 409 | `{"erro":"bilhete_nao_aberto"}` |
| Abrir bilhete com placa ocupada | 409 | `{"erro":"bilhete_em_aberto"}` |

> [!IMPORTANT]
> Precedência: 422 (formato) → 404 (existência) → 409 (estado). Rotas ou métodos inexistentes devolvem 404/405 também no formato `{"erro": "..."}` (use `rota_nao_encontrada` e `metodo_nao_permitido`).

## Requisitos não funcionais

- O serviço escuta na porta 8001 dentro do container (ver `plan.md` para a porta auxiliar) e responde `Content-Type: application/json`.
- Estado em memória; reiniciar o processo zera tudo. Operações de abrir, encerrar e cancelar são atômicas (lock).
- O repositório gerado entrega `Dockerfile`, `README.md`, `requirements.txt` e testes próprios (detalhes em `plan.md`).