
# Plan — Arquitetura e decisões

## Stack

Python 3.11 + FastAPI + uvicorn, persistência em memória, testes com pytest + httpx (`TestClient`)

## Decisões técnicas (com justificativa)

| # | Decisão | Justificativa |
| --- | --- | --- |
| D1 | Dinheiro em centavos inteiros, divisão `//`, nunca `float` | Ponto flutuante acumula erro (0.1 + 0.2 ≠ 0.3); a suíte testa isso[^float]. |
| D2 | `minutos` = truncamento de `(saida − entrada)` para minutos inteiros | A suíte abre bilhetes com `entrada` no passado e encerra "agora", então a duração sempre vem com milissegundos a mais; arredondar para cima transformaria "exatamente 30 min" em 31 e cobraria a fração errada[^trunc]. |
| D3 | Frações por `(minutos + 29) // 30`; valor da fração `= 400 * 30 // 60 = 200` | Aritmética inteira sem erro; a mesma fórmula serve se a variante mudar. |
| D4 | Tolerância avaliada antes da fração: `minutos <= 10 → 0`; acima disso cobra do minuto zero | É o contrato (UC7): tolerância não é desconto. |
| D5 | Teto aplicado por último com `min(valor, 5000)` | Garante que nenhum caminho de cálculo ultrapasse o teto. |
| D6 | Relógio isolado em `clock.now()` | Permite fixar o instante nos testes próprios com `monkeypatch`, sem esperar tempo real. |
| D7 | Corpo lido com `await request.json()` em `try/except`, **validação manual** (sem modelos pydantic) | O FastAPI/pydantic devolve `{"detail": ...}` e 422 próprios; validar manualmente garante `{"erro": ...}` e a precedência 422 → 404 → 409. |
| D8 | Handlers globais para `AppError`, `RequestValidationError`, `StarletteHTTPException` e `Exception` | Todo erro sai como `{"erro": "<codigo>"}`; nenhum 500 vaza `detail`. |
| D9 | Parâmetro de caminho `{id}` tratado como **string**; se não for dígitos → 404 `bilhete_nao_encontrado` | Evita 422 automático do framework para `/bilhetes/abc/encerramento`. |
| D10 | Rota `GET /bilhetes/ativos` declarada **antes** de qualquer rota parametrizada de `/bilhetes` | Evita captura de `ativos` como parâmetro. |
| D11 | Índice `placa → id do bilhete aberto` no repositório | Checagem de UC8 em O(1) e atômica sob o mesmo lock. |
| D12 | Ordenação de listas por `(entrada, id)` **decrescente** | Define "mais recentes primeiro" de forma determinística, inclusive com `entrada` retroativa. |
| D13 | Data do relatório = **data da `entrada` em `-03:00`** | A `saida` é sempre "agora" e não é controlável pela suíte; a `entrada` é o gancho de testabilidade[^dia]. |
| D14 | Média do relatório com inteiros: `(2*soma + n) // (2*n)` | Implementa "0,5 arredonda para cima" sem `round()` (que faz arredondamento bancário no Python). |
| D15 | Validação da `entrada` por regex `^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}(:\d{2}(\.\d+)?)?(Z|[+-]\d{2}:\d{2})?$` seguida de `datetime.fromisoformat` | Rejeita formatos que o Python 3.11 aceitaria mas o contrato não (ex.: só data). Sem fuso → `-03:00`. |

[^float]: Em ponto flutuante, `0.1 + 0.2` dá `0.30000000000000004`. Em centavos inteiros a soma e a comparação são exatas.
[^trunc]: Exemplo: `entrada = agora − 30 min`; ao encerrar, a duração real é `30 min + 0,02 s`. Truncando, `minutos = 30` → 1 fração (200). Arredondando para cima, `minutos = 31` → 2 frações (400), o que reprova o teste de "fração exata".
[^dia]: Se a `saida` definisse o dia, um bilhete aberto com `entrada` em data passada cairia sempre no dia de hoje e seria impossível testar o relatório de datas específicas.


> [!NOTE]
> `PORTA_SERVICO = 8001` é a porta documentada no README e em `EXPOSE`; 8080 é auxiliar. O Dockerfile faz `EXPOSE 8001 8080`.

## Fluxo de geração esperado

1. Criar `config.py`, `clock.py`, `errors.py`, `pricing.py` (base sem dependência de HTTP).
2. Criar `store.py` e `service.py`, depois `main.py` e `server.py`.
3. Escrever os testes de `tests.md`, rodar `pytest` e corrigir até passar tudo.
4. Criar `Dockerfile`, `Containerfile`, `.dockerignore`, `.gitignore`, `requirements.txt`, `README.md`.