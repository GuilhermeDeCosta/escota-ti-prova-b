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
