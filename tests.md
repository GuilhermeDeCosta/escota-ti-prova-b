
# Tests — Cenários de teste (TDD)

Cada linha das tabelas vira uma função def test_... própria em tests/ (sem parametrize que junte linhas). Total: 56 cenários. Todos os valores usam a variante: tarifa 400, fração 30, teto 5000, tolerância 10 (fração = 200 centavos).

Como controlar o tempo: os testes de API fixam clock.now() (via monkeypatch) em 2026-10-12T12:00:00-03:00 e abrem bilhetes com entrada = agora − N minutos. Os testes de pricing chamam compute_price(minutes) direto.

> [!WARNING]
> Teto: os testes P12 a P14 e A09 verificam que valor_centavos nunca passa de 5000. Tolerância: P02 a P04 verificam que 11 minutos cobra 200 (integral), nunca 0 nem "1 minuto".

## Grupo P — Cobrança (tests/test_pricing.py)

| # | Entrada (minuto`) | Esperado (valor_centavos) | Regra / tipo |
| --- | --- | --- | --- |
| P01 | 0 | 0 | tolerância, borda |
| P02 | 10 | 0 | tolerância (limite exato), borda |
| P03 | 11 | 200 | tolerância +1 min cobra integral, borda |
| P04 | 1 | 0 | tolerância, feliz |
| P05 | 30 | 200 | fração exata cobra 1 fração, borda |
| P06 | 31 | 400 | +1 min cobra a fração seguinte, borda |
| P07 | 60 | 400 | hora cheia = tarifa, feliz |
| P08 | 61 | 600 | +1 min após hora cheia, borda |
| P09 | 95 | 800 | arredondamento para cima (4 frações), feliz |
| P10 | 90 | 600 | fração exata (3 frações), borda |
| P11 | 120 | 800 | feliz |
| P12 | 750 | 5000 | teto atingido exatamente, borda |
| P13 | 751 | 5000 | teto (+1 min) não ultrapassa, borda |
| P14 | 4320 (3 dias) | 5000 | teto, borda |
| P15 | compute_minutes com saida − entrada = 30 min + 59 s | 30 | truncamento, borda |
| P16 | compute_minutes com entrada no futuro | 0 | nunca negativo, borda |

## Grupo A — API de bilhetes (tests/test_api.py)

| # | Cenário | Esperado | Tipo |
| --- | --- | --- | --- |
| A01 | `POST /bilhetes` placa `ABC1D23` | `201`, `id=1`, `status="aberto"`, `entrada` com `-03:00` | feliz |
| A02 | `POST /bilhetes` com `entrada=2026-10-12T08:30:00-03:00` | `201`, `entrada` idêntica | feliz |
| A03 | `POST /bilhetes` com `entrada=2026-10-12T11:30:00Z` | `201`, `entrada=2026-10-12T08:30:00-03:00` | borda |
| A04 | Placa minúscula `abc1d23` | `422` `placa_invalida` | borda |
| A05 | Placa com 6 caracteres `ABC1D2` e com 8 `ABC1D234` | `422` `placa_invalida` | borda |
| A06 | Sem `placa` / body vazio / body não-JSON | `422` `placa_invalida` | borda |
| A07 | `entrada="ontem"` | `422` `entrada_invalida` | borda |
| A08 | Encerrar bilhete aberto há 95 min | `200`, `minutos=95`, `valor_centavos=800`, `type(valor_centavos) is int` | feliz |
| A09 | Encerrar bilhete aberto há 3 dias | `200`, `valor_centavos=5000` | borda |
| A10 | Encerrar bilhete aberto há exatos 30 min | `valor_centavos=200` | borda |
| A11 | Encerrar bilhete aberto há 31 min | `valor_centavos=400` | borda |
| A12 | Encerrar bilhete aberto há 10 min | `valor_centavos=0` | borda |
| A13 | Encerrar bilhete aberto há 11 min | `valor_centavos=200` | borda |
| A14 | Resposta do encerramento tem exatamente as 6 chaves do contrato (sem `valor`, sem `status`) | conjunto de chaves igual | feliz |
| A15 | Encerrar `id` inexistente (`999`) e não numérico (`abc`) | `404` `bilhete_nao_encontrado` | borda |
| A16 | Encerrar duas vezes o mesmo bilhete | 2ª: `409` `bilhete_ja_encerrado` | borda |
| A17 | Encerrar bilhete cancelado | `409` `bilhete_ja_encerrado` | borda |
| A18 | Cancelar bilhete aberto | `200`, `status="cancelado"`, sem `saida`/`valor_centavos` | feliz |
| A19 | Cancelar `id` inexistente | `404` `bilhete_nao_encontrado` | borda |
| A20 | Cancelar bilhete encerrado e cancelar duas vezes | `409` `bilhete_nao_aberto` | borda |
| A21 | `GET /bilhetes/ativos` com 2 abertos, 1 encerrado, 1 cancelado | só os 2 abertos | feliz |
| A22 | `GET /bilhetes/ativos` com entradas 10:00 e 11:00 | o de 11:00 primeiro | borda |
| A23 | `GET /bilhetes/ativos` sem bilhetes | `200` `[]` | borda |
| A24 | `GET /bilhetes?placa=` com 1 aberto, 1 encerrado, 1 cancelado | 3 itens, mais recente primeiro | feliz |
| A25 | `GET /bilhetes?placa=` de placa válida sem histórico | `200` `[]` | borda |
| A26 | `GET /bilhetes?placa=abc` e `GET /bilhetes` sem placa | `422` `placa_invalida` | borda |
| A27 | `POST` de placa com bilhete aberto | `409` `bilhete_em_aberto` | borda |
| A28 | Reabrir placa após encerrar | `201` com novo `id` | borda |
| A29 | Reabrir placa após cancelar | `201` com novo `id` | borda |
| A30 | Placa A aberta e `POST` da placa B | `201` | borda |
| A31 | Placa ocupada com `entrada` inválida | `422` `entrada_invalida` (e não 409) | borda |
| A32 | Qualquer erro devolve corpo `{"erro": "..."}` sem chave `detail` | chaves = `{"erro"}` | borda |

## Grupo R — Relatório diário (`tests/test_report.py`)

| # | Cenário | Esperado | Tipo |
| --- | --- | --- | --- |
| R01 | Encerrados com 10 e 11 min no dia | `tempo_medio_minutos=11` (10,5 → 11) | borda |
| R02 | Encerrados com 10, 10 e 11 min | `tempo_medio_minutos=10` (10,33 → 10) | borda |
| R03 | 1 encerrado (400), 1 aberto, 1 cancelado, mesmo dia | `total_bilhetes=3`, `faturamento_centavos=400` | feliz |
| R04 | Bilhete de outro dia | não entra em nenhum campo | borda |
| R05 | `data=2026-13-01`, `data=05/10/2026` e sem `data` | `422` `data_invalida` | borda |
| R06 | Dia sem bilhetes | `200`, `total_bilhetes=0`, `faturamento_centavos=0`, `tempo_medio_minutos=0` | borda |
| R07 | `data=2026-02-30` (inexistente) | `422` `data_invalida` | borda |
| R08 | Faturamento de 2 encerrados (200 e 5000) | `faturamento_centavos=5200` (inteiro) | feliz |
