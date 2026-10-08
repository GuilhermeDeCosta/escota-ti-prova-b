## Tasks — Decomposição
Execute na ordem das dependências Cada tarefa só termina quando seu critério de pronto for verdadeiro Regras gerais em constitution.md, comportamento em spec.md, decisões em plan.md, cenários em tests.md

[ ] T1 — Scaffolding: criar pacote app/ com config.py (400, 30, 5000, 10, 8001), clock.py e errors.py (AppError + handlers com corpo {"erro": ...}) Pronto quando: importar os módulos não gera erro e as constantes batem com a constitution.md  
[ ] T2 — Cobrança pura (T1): app/pricing.py com compute_minutes (truncamento, nunca negativo) e compute_price (tolerância → frações → teto, só inteiros) Pronto quando: os cenários P01 a P16 passam  
[ ] T3 — Repositório (T1): app/store.py com Lock, ids sequenciais a partir de 1 e índice placa → bilhete aberto Pronto quando: ids nunca se repetem e a placa libera após encerrar/cancelar  
[ ] T4 — UC1 abrir bilhete (T2, T3): validação manual de placa (^[A-Z0-9]{7}$) e entrada (regex + fromisoformat, conversão para -03:00), POST /bilhetes Pronto quando: A01 a A07 passam  
[ ] T5 — UC8 uma vaga por placa (T4): 409 bilhete_em_aberto depois das validações 422 Pronto quando: A27 a A31 passam  
[ ] T6 — UC2 encerrar (T4): POST /bilhetes/{id}/encerramento com {id} string, 404 antes de 409, resposta com as 6 chaves Pronto quando: A08 a A17 passam  
[ ] T7 — UC5 cancelar (T4): POST /bilhetes/{id}/cancelamento, sem saida/valor_centavos Pronto quando: A18 a A20 passam  
[ ] T8 — UC3 e UC6 listagens (T4, T6, T7): GET /bilhetes/ativos (declarada antes de rotas parametrizadas) e GET /bilhetes?placa=, ordenação (entrada, id) decrescente Pronto quando: A21 a A26 passam  
[ ] T9 — UC4 relatório diário (T6, T7): GET /relatorios/diario?data=, dia pela data da entrada em -03:00, média com (2*soma + n) // (2*n) Pronto quando: R01 a R08 passam  
[ ] T10 — Erros planos (T1): handlers para validação, 404, 405 e exceções, sem chave detail Pronto quando: A32 passa e nenhuma entrada inválida gera 500  
[ ] T11 — Entrypoint e portas (T4): app/server.py escutando em 8001 e 8080 no mesmo processo (snippet do plan.md) Pronto: GET /bilhetes/ativos responde 200 em ambas as portas  
[ ] T12 — Testes próprios (T2 a T10): tests/test_pricing.py, tests/test_api.py, tests/test_report.py com uma função por linha de tests.md (≥ 56) Pronto quando: pytest verde  
[ ] T13 — SDLC e entrega (T11, T12): Dockerfile (+ Containerfile idêntico), requirements.txt fixado, .gitignore, .dockerignore, README.md com docker, podman, uvicorn e pytest Pronto quando: docker build e docker run -p 8001:8001 sobem o serviço e o ruff check não acusa nada  
[ ] T14 — Revisão final (T13): rodar a suíte completa, conferir a tabela de cobrança da spec.md, garantir ausência de segredos, print, imports soltos e arquivos de cache Pronto quando: tudo verde e a árvore limpa  

> [!IMPORTANT]
> Antes de encerrar: conferir que valor_centavos é sempre int, que o teto é 5000, que 11 minutos cobra 200 e que nenhum erro usa detail