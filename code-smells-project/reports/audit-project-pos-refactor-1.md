================================
ARCHITECTURE AUDIT REPORT
================================
Project: code-smells-project
Stack:   Python + Flask 3.1.1
Files:   19 analyzed | ~900 lines of code

## Summary
CRITICAL: 3 | HIGH: 3 | MEDIUM: 3 | LOW: 3

## Findings

### [CRITICAL] SQL Injection
File: routes/admin_routes.py:34
Description: O endpoint `/admin/query` executa SQL arbitrario recebido no body da requisicao. A query do usuario (`dados.get("sql", "")`) e passada diretamente para `cursor.execute(query)` sem nenhuma sanitizacao ou uso de parametros preparados. Qualquer detentor do token admin pode executar SQL arbitrario incluindo DROP, UPDATE e DELETE.
Impacto: Um atacante com o token admin pode destruir todo o banco, ler dados sensiveis (_hashes de senha_), ou modificar registros arbitrariamente.
Recommendation: Remover o endpoint de execucao de SQL arbitrario. Se necessario para debug, restringir a queries SELECT read-only e implementar um allowlist de queries pre-definidas.

### [CRITICAL] Hardcoded Credentials / Secrets
File: config/settings.py:3,6
Description: Variaveis de configuracao possuem fallbacks inseguros que podem facilmente ir para producao:
- `SECRET_KEY = os.environ.get("SECRET_KEY", "dev-secret-key-change-in-production")` (linha 3)
- `ADMIN_TOKEN = os.environ.get("ADMIN_TOKEN", "admin-dev-token")` (linha 6)
O token admin padrao `"admin-dev-token"` e trivialmente adivinhavel. A secret key padrao e um placeholder conhecido.
Impacto: Se as variaveis de ambiente nao forem configuradas em producao, o sistema fica com credenciais fracas e previsiveis, permitindo acesso nao autorizado aos endpoints admin e sessoes forjadas.
Recommendation: Remover os fallbacks inseguros. Lançar excecao na inicializacao se as variaveis nao estiverem definidas em producao, ou usar valores aleatorios apenas em modo dev explicito.

### [CRITICAL] Unauthenticated Sensitive Endpoints
File: routes/product_routes.py, routes/user_routes.py, routes/order_routes.py
Description: Todos os endpoints CRUD de produtos, usuarios e pedidos nao possuem nenhuma autenticacao. Qualquer usuario anonimo pode:
- Criar, alterar e deletar produtos (product_routes.py)
- Criar usuarios e listar todos os usuarios (user_routes.py)
- Criar pedidos e alterar status de pedidos (order_routes.py:30-35 — `PUT /pedidos/<id>/status`)
Apenas os endpoints `/admin/*` possuem protecao via `@require_admin`.
Impacto: Qualquer consumidor da API pode realizar operacoes destrutivas (deletar produtos, alterar status de pedidos, criar usuarios admin) sem autenticacao.
Recommendation: Implementar autenticacao via JWT ou sessoes para todos os endpoints mutaveis (POST, PUT, DELETE). Operacoes administrativas devem exigir role admin.

---

### [HIGH] Business Logic in Route Layer
File: routes/admin_routes.py:12-18, 26-43
Description: Os handlers `reset_database()` (linhas 12-18) e `executar_query()` (linhas 26-43) acessam o banco de dados diretamente via `get_db()`, bypassando completamente as camadas de model e controller. Toda a logica de limpeza de tabelas e execucao de queries esta implementada no route handler.
Codigo encontrado:
```python
db = get_db()
cursor = db.cursor()
cursor.execute("DELETE FROM itens_pedido")
cursor.execute("DELETE FROM pedidos")
...
```
Impacto: Viola a separacao MVC estabelecida no projeto. A logica de dados nos routes torna impossivel testar unitariamente ou reutilizar essas operacoes.
Recommendation: Mover as operacoes de reset e query para um model (`admin_model.py`) e criar um controller para orquestrar a chamada. O route deve apenas extrair dados do request e chamar o controller.

### [HIGH] Sensitive Data Exposure in API Responses
File: routes/admin_routes.py:34-38
Description: O endpoint `/admin/query` retorna resultados diretos do banco sem filtragem. Um atacante com o token admin pode executar `SELECT * FROM usuarios` e obter todos os hashes de senha, ou `SELECT senha FROM usuarios` para leitura direta de credenciais hashed.
Codigo encontrado:
```python
rows = cursor.fetchall()
result = [dict(row) for row in rows]
return jsonify({"dados": result, "sucesso": True}), 200
```
Impacto: Hashes de senha podem ser extraidos e submetidos a ataques de forca bruta offline. Exposicao de dados sensiveis viola principios de seguranca.
Recommendation: Remover o endpoint de query arbitraria. Se retenido para debug, implementar filtragem de colunas sensiveis (senha, hash) nos resultados.

### [HIGH] Tight Coupling — Routes Import Database Directly
File: routes/admin_routes.py:3
Description: O arquivo de rotas importa diretamente o modulo de banco de dados (`from database.connection import get_db`), quebrando a direcao de dependencia MVC estabelecida no projeto (`Routes -> Controllers -> Models`). As demais rotas corretamente importam apenas controllers.
Codigo encontrado:
```python
from database.connection import get_db
```
Impacto: Viola a arquitetura MVC. Mudancas na camada de banco afetam diretamente as rotas. Dificulta testes com mocks.
Recommendation: Remover o import direto de `get_db` das rotas. Criar `models/admin_model.py` e `controllers/admin_controller.py` para encapsular as operacoes admin, seguindo o padrao das demais entidades.

---

### [MEDIUM] Multiple Queries Instead of Aggregation (Inefficient DB Access)
File: models/report_model.py:8-14
Description: O metodo `sales_report()` executa 5 queries separadas para contar pedidos por status, quando uma unica query com `GROUP BY` resolveria todas as contagens em uma unica operacao.
Codigo encontrado:
```python
total_pedidos = db.execute("SELECT COUNT(*) FROM pedidos").fetchone()[0]
pendentes = db.execute("SELECT COUNT(*) FROM pedidos WHERE status = 'pendente'").fetchone()[0]
aprovados = db.execute("SELECT COUNT(*) FROM pedidos WHERE status = 'aprovado'").fetchone()[0]
cancelados = db.execute("SELECT COUNT(*) FROM pedidos WHERE status = 'cancelado'").fetchone()[0]
```
Impacto: 5 roundtrips ao banco para dados que poderiam ser obtidos em 1. Desperdicio de recursos que escala com o uso.
Recommendation: Substituir as 5 queries por uma unica query: `SELECT status, COUNT(*) as count FROM pedidos GROUP BY status` e processar os resultados em Python.

### [MEDIUM] Inadequate Error Handling — Raw Exception Exposure
File: routes/admin_routes.py:43
Description: O bloco `except` no endpoint de query expoe a mensagem bruta da excecao para o cliente: `return jsonify({"erro": str(e)}), 500`. Mensagens de erro do SQLite podem conter nomes de tabelas, colunas e caminhos do filesystem.
Codigo encontrado:
```python
except Exception as e:
    return jsonify({"erro": str(e)}), 500
```
Impacto: Vazamento de detalhes internos da aplicacao (estrutura do banco, caminhos de arquivo) para o cliente. Facilita reconnaissance por atacantes.
Recommendation: Logar o erro completo internamente e retornar uma mensagem generica ao cliente, seguindo o padrao do `handle_errors` middleware.

### [MEDIUM] Missing Input Validation
File: routes/product_routes.py:23-26
Description: Os parametros `preco_min` e `preco_max` sao convertidos com `float()` sem tratamento de excecao. Um valor invalido como `?preco_min=abc` causara um `ValueError` que sera capturado genericamente pelo `handle_errors` middleware, retornando "Erro interno do servidor" em vez de uma mensagem util.
Codigo encontrado:
```python
if preco_min:
    preco_min = float(preco_min)
if preco_max:
    preco_max = float(preco_max)
```
Impacto: Mensagem de erro generica confunde o consumidor da API. Erro de validacao de input retorna status 500 em vez de 400.
Recommendation: Envolver a conversao em try/except com retorno de erro 400 claro (ex: "preco_min deve ser um numero valido").

---

### [LOW] Debug Artifacts in Production Code
File: app.py:53-56
Description: Mensagens de startup usam `print()` em vez do logger configurado na linha 1.
Codigo encontrado:
```python
print("=" * 50)
print("SERVIDOR INICIADO")
print("Rodando em http://localhost:5000")
print("=" * 50)
```
Impacto: Saida de print nao e gerenciavel por configuracao de logging e polui stdout em producao.
Recommendation: Substituir por `logger.info()` usando o logger ja importado no topo do arquivo.

### [LOW] Magic Numbers / Magic Strings
File: app.py:57, report_controller.py:12, order_model.py:21,61
Description: Valores literais espalhados pelo codigo sem constantes nomeadas:
- `app.py:57` — porta `5000` hardcoded
- `report_controller.py:12` — versao `"1.0.0"` hardcoded
- `order_model.py:61` — status `'pendente'` hardcoded no INSERT
- `order_model.py:21` — string `'Desconhecido'` como fallback para produto sem nome
Impacto: Dificil alterar esses valores de forma consistente. A versao da API precisa ser atualizada em multiplos lugares.
Recommendation: Mover porta e versao para `config/settings.py`. Usar as constantes de `settings.VALID_ORDER_STATUSES` para o status inicial do pedido.

### [LOW] Unused Code
File: models/product_model.py:87-93
Description: A funcao `update_stock()` e definida no product_model mas nunca e chamada em nenhum lugar do projeto. A atualizacao de estoque e feita inline em `order_model.py:73-76`.
Codigo encontrado:
```python
def update_stock(product_id, quantity):
    db = get_db()
    db.execute(
        "UPDATE produtos SET estoque = estoque - ? WHERE id = ?",
        [quantity, product_id],
    )
```
Impacto: Codigo morto polui o codebase e causa confusao sobre qual funcao usar para atualizar estoque.
Recommendation: Remover `update_stock()` do product_model ou refatorar `order_model.py` para usa-la em vez de duplicar a logica de atualizacao de estoque.

================================
Total: 12 findings
================================