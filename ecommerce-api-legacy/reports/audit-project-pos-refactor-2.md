================================
ARCHITECTURE AUDIT REPORT
================================
Project: ecommerce-api-legacy
Stack:   JavaScript (Node.js) + Express 4.18.2
Files:   19 analyzed | ~400 lines of code

## Summary
CRITICAL: 1 | HIGH: 3 | MEDIUM: 3 | LOW: 3

## Findings

### [CRITICAL] Unauthenticated Sensitive Endpoints
File: src/routes/checkoutRoutes.js, src/routes/reportRoutes.js, src/routes/userRoutes.js
Description: Todos os 3 endpoints da API nao possuem nenhuma autenticacao:
- `POST /api/checkout` — qualquer usuario anonimo pode processar checkouts e pagamentos
- `GET /api/admin/financial-report` — endpoint administrativo de relatorio financeiro acessivel sem autenticacao
- `DELETE /api/users/:id` — qualquer usuario anonimo pode deletar qualquer conta de usuario

A rota `/api/admin/financial-report` e particularmente critica por ser explicitamente administrativa sem nenhuma protecao. O endpoint `DELETE /api/users/:id` permite destruicao irreversivel de dados por qualquer consumidor da API.
Impacto: Qualquer pessoa com acesso a API pode deletar todos os usuarios, acessar relatorios financeiros e processar checkouts fraudulentos.
Recommendation: Implementar middleware de autenticacao (JWT ou sessao). Proteger rotas admin com verificacao de role. Adicionar autenticacao em todos os endpoints mutaveis (POST, DELETE).

---

### [HIGH] Hardcoded Default Password
File: src/controllers/checkoutController.js:21
Description: Quando o campo `password` nao e fornecido no checkout, o usuario e criado com a senha default `'123456'`: `user = await userModel.create(db, userName, email, password || '123456')`. Esta senha fraca e previsivel e atribuida sem o conhecimento do usuario.
Codigo encontrado:
```javascript
user = await userModel.create(db, userName, email, password || '123456');
```
Impacto: Usuarios criados sem senha recebem uma senha fraca e conhecida. Atacantes podem tentar login com credenciais preveiveis em contas criadas via checkout sem senha.
Recommendation: Exigir que a senha seja obrigatoria no checkout, ou gerar uma senha aleatoria segura e enviar por email. Nunca usar senha hardcoded como fallback.

### [HIGH] Sensitive Data Exposure — Password Hash in Query Result
File: src/models/userModel.js:5
Description: A funcao `findByEmail` retorna o campo `pass` (hash da senha) no resultado: `SELECT id, name, email, pass FROM users WHERE email = ?`. O controller `checkoutController.js:19` chama esta funcao para verificar se o usuario ja existe, recebendo o hash no objeto retornado. Embora o hash nao seja retornado na resposta HTTP, ele esta disponivel no objeto `user` em memoria, criando risco de exposicao acidental.
Codigo encontrado:
```javascript
function findByEmail(db, email) {
  return get(db, 'SELECT id, name, email, pass FROM users WHERE email = ?', [email]);
}
```
Impacto: Se o objeto `user` for acidentalmente incluido em uma resposta de log ou erro, o hash e exposto. O controller nunca precisa do hash neste fluxo.
Recommendation: Remover `pass` da query SELECT em `findByEmail`. Se a autenticacao futura precisar do hash, criar uma funcao separada `findByEmailWithHash`.

### [HIGH] Tight Coupling — Controller Bypasses Model Layer
File: src/controllers/userController.js:3,13
Description: O controller importa `all` diretamente de `database/connection` e executa uma query na tabela `enrollments`, bypassando completamente o `enrollmentModel`: `const enrollments = await all(db, 'SELECT id FROM enrollments WHERE user_id = ?', [userId])`. O proprio `enrollmentModel` ja possui um metodo `deleteByUserId`, mas o controller nao usa o model para obter os IDs.
Codigo encontrado:
```javascript
const { all } = require('../database/connection');
...
const enrollments = await all(db, 'SELECT id FROM enrollments WHERE user_id = ?', [userId]);
```
Impacto: Viola a separacao em camadas MVC. O controller acessa o banco diretamente, acoplando-se a estrutura da tabela `enrollments`. Se a tabela for renomeada ou a query mudar, o controller precisa ser alterado em vez de apenas o model.
Recommendation: Adicionar funcao `findByUserId(db, userId)` ao `enrollmentModel` e usa-la no controller. Remover o import direto de `all` do database no controller.

---

### [MEDIUM] Unused Configuration Variables
File: src/config/index.js:6-10
Description: Cinco variaveis de ambiente sao lidas na configuracao mas nunca utilizadas em nenhum lugar do codigo:
- `paymentGatewayKey` (linha 6) — nao e usada no `paymentService.js`, que faz validacao local de cartao
- `smtpUser` (linha 7) e `smtpPass` (linha 8) — nao existe servico de email implementado
- `dbUser` (linha 9) e `dbPass` (linha 10) — SQLite nao usa autenticacao
Codigo encontrado:
```javascript
paymentGatewayKey: process.env.PAYMENT_GATEWAY_KEY || '',
smtpUser: process.env.SMTP_USER || '',
smtpPass: process.env.SMTP_PASS || '',
dbUser: process.env.DB_USER || '',
dbPass: process.env.DB_PASS || '',
```
Impacto: Configuracoes mortas confundem desenvolvedores sobre quais variaveis de ambiente sao realmente necessarias. O `paymentGatewayKey` sugere integracao com gateway real que nao existe.
Recommendation: Remover as variaveis nao utilizadas ou implementar a integracao correspondente (payment gateway, SMTP).

### [MEDIUM] No Duplicate Enrollment Check
File: src/controllers/checkoutController.js:29
Description: O fluxo de checkout nao verifica se o usuario ja esta matriculado no curso antes de criar uma nova matricula: `const enrollment = await enrollmentModel.create(db, user.id, courseId)`. Se o mesmo usuario fizer checkout do mesmo curso duas vezes, serao criadas matriculas duplicadas e cobrancas multiplas.
Codigo encontrado:
```javascript
const enrollment = await enrollmentModel.create(db, user.id, courseId);
```
Impacto: Cobranca duplicada para o mesmo curso. Dados de matricula inconsistentes. Relatorio financeiro inflado com pagamentos duplicados.
Recommendation: Verificar se ja existe matricula antes de criar: `const existing = await enrollmentModel.findByUserAndCourse(db, user.id, courseId)`. Se existir, retornar erro ou reutilizar a matricula existente.

### [MEDIUM] No Input Format Validation
File: src/controllers/checkoutController.js:9-12
Description: O checkout valida apenas a presenca dos campos obrigatorios, mas nao valida o formato de nenhum deles:
- `email` — qualquer string e aceita, sem verificacao de formato
- `courseId` — nao e convertido para numero, pode receber strings nao-numericas
- `cardNumber` — nao e validado por tamanho ou formato, apenas por comecar com '4'
- `userName` — aceita strings vazias se whitespace for enviado
Codigo encontrado:
```javascript
if (!userName || !email || !courseId || !cardNumber) {
  throw new AppError('Missing required fields: userName, email, courseId, cardNumber', 400, true);
}
```
Impacto: Dados invalidos podem causar erros de banco (courseId nao-numerico) ou comportamento inesperado (email invalido, cartao com formato errado).
Recommendation: Adicionar validacao de formato para cada campo: regex de email, verificacao numerica de courseId, validacao de tamanho de cardNumber, trim de userName.

---

### [LOW] Dead Code — Unused Function
File: src/models/courseModel.js:7-9
Description: A funcao `findAll(db)` e exportada mas nunca chamada em nenhum lugar do projeto. Nenhuma rota ou controller a utiliza.
Codigo encontrado:
```javascript
function findAll(db) {
  return all(db, 'SELECT id, title, price, active FROM courses', []);
}
```
Impacto: Codigo morto polui o codebase e causa confusao sobre a API disponivel.
Recommendation: Remover a funcao ou implementa-la em um endpoint de listagem de cursos.

### [LOW] Magic Numbers / Magic Strings
File: src/services/paymentService.js:7, src/controllers/checkoutController.js:21, src/database/schema.js:13, src/models/userModel.js:13
Description: Valores literais espalhados pelo codigo sem constantes nomeadas:
- `paymentService.js:7` — `cardNumber.startsWith('4')` — regra de aprovacao Visa sem constante ou comentario
- `checkoutController.js:21` — `'123456'` senha default hardcoded
- `schema.js:13` — `'123'` senha de seed hardcoded
- `userModel.js:13` — `bcrypt.hash(plainPassword, 10)` — salt rounds `10` sem constante
Impacto: Regras de negocio dispersas em valores literais. Dificil ajustar sem buscar em todo o codigo.
Recommendation: Extrair para constantes nomeadas em `config` ou no topo do modulo: `BCRYPT_SALT_ROUNDS = 10`, `DEFAULT_PASSWORD`, `SEED_PASSWORD`.

### [LOW] Debug Artifacts in Production Code
File: src/app.js:20, src/middlewares/errorHandler.js:2
Description: Uso de `console.log` e `console.error` como mecanismo de logging em producao:
- `app.js:20` — `console.log('LMS API running on port ${config.port}')`
- `errorHandler.js:2` — `console.error('[ERROR]', err.message || err)`
Impacto: Saida de console nao e gerenciavel por configuracao de logging. Sem niveis, timestamps ou contexto estruturado.
Recommendation: Usar uma biblioteca de logging estruturado (winston, pino) ou ao menos abstrair em um logger configuravel.

================================
Total: 10 findings
================================