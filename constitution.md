# Constitution — Regras persistentes do projeto

> [!WARNING] Estas regras valem para todo arquivo gerado e para qualquer decisão não coberta explicitamente em `spec.md`, `plan.md` ou `tests.md`.

## Parâmetros da variante (valores fixos deste projeto)

| Parâmetro | Valor | Observação |
| --- | --- | --- |
| `TARIFA_HORA_CENTAVOS` | **400** | hora cheia |
| `FRACAO_MINUTOS` | **30** | granularidade de cobrança |
| `TETO_DIARIO_CENTAVOS` | **5000** | máximo por bilhete |
| `TOLERANCIA_MINUTOS` | **10** | minutos grátis por bilhete |
| `PORTA_SERVICO` | **8001** | porta do serviço |
| valor da fração (derivado) | **200** | `400 ÷ (60 ÷ 30)` |

Regras operacionais
1. Nomes do contrato são literais em português: rotas (/bilhetes, /bilhetes/ativos, /relatorios/diario), chaves JSON (placa, entrada, saida, minutos, valor_centavos, status, total_bilhetes, faturamento_centavos, tempo_medio_minutos) e códigos de erro (placa_invalida etc.). Identificadores internos (funções, classes, variáveis, arquivos) em inglês; documentação (README, docstrings) em português.
2. Dinheiro é sempre inteiro em centavos. Proibido float em qualquer cálculo ou resposta; usar divisão inteira //. A API nunca retorna número com ponto decimal.
3. Corpo de erro é sempre plano: {"erro": "<codigo>"}. Proibido o formato padrão {"detail": ...} do framework — todos os handlers de erro (inclusive validação e 404/405 padrão) devem ser sobrescritos.
4. Precedência de erros: validação de formato (422) → existência (404) → regra de estado (409). Um payload malformado nunca dispara 409.
5. Todo endpoint documenta e trata seus status de erro (tabela em spec.md); nenhum caminho pode terminar em 500 por entrada do cliente.
6. Datas e horas: sempre ISO-8601 com fuso -03:00 e precisão de segundos (ex.: 2026-10-12T08:30:00-03:00). Entradas com outro fuso são convertidas para -03:00.
7. Sem variáveis de ambiente obrigatórias: os parâmetros da variante ficam como constantes em config.py; o serviço sobe com docker run puro.
8. Stack e dependências: Python 3.11, FastAPI, uvicorn; testes com pytest e httpx. Nenhuma outra dependência. Persistência em memória (sem banco).
9. Higiene de código 
10. Higiene de repositório: nenhum segredo, token ou senha no código
11. Todo bloco de código em .md tem no máximo 20 linhas, especificar, nunca implementar.