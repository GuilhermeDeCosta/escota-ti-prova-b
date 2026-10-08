
# Tests — Cenários de teste (TDD)

Cada linha das tabelas vira uma função `def test_...` própria em `tests/` (sem `parametrize` que junte linhas). Total: 56 cenários. Todos os valores usam a variante: tarifa 400, fração 30, teto 5000, tolerância 10 (fração = 200 centavos).

Como controlar o tempo: os testes de API fixam `clock.now()` (via `monkeypatch`) em `2026-10-12T12:00:00-03:00` e abrem bilhetes com `entrada = agora − N minutos`. Os testes de `pricing` chamam `compute_price(minutes)` direto.

> [!WARNING]
> Teto: os testes P12 a P14 e A09 verificam que `valor_centavos` nunca passa de 5000**. Tolerância: P02 a P04 verificam que 11 minutos cobra 200 (integral), nunca 0 nem "1 minuto".

## Grupo P — Cobrança (`tests/test_pricing.py`)

| # | Entrada (`minutos`) | Esperado (`valor_centavos`) | Regra / tipo |
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
| P15 | `compute_minutes` com `saida − entrada = 30 min + 59 s` | 30 | truncamento, borda |
| P16 | `compute_minutes` com `entrada` no futuro | 0 | nunca negativo, borda |
