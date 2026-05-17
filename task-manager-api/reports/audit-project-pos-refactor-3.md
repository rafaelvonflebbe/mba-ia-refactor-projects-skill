================================
ARCHITECTURE AUDIT REPORT
================================
Project: task-manager-api
Stack:   Python + Flask 3.0.0
Files:   23 analyzed | ~1024 lines of code

## Summary
CRITICAL: 2 | HIGH: 1 | MEDIUM: 3 | LOW: 2

## Findings

### [CRITICAL] Unauthenticated Sensitive Endpoints
File: routes/task_routes.py, routes/user_routes.py, routes/category_routes.py, routes/report_routes.py
Description: All CRUD endpoints — including DELETE users, DELETE tasks, role assignment, and user creation — have zero authentication or authorization middleware. The login endpoint at `controllers/user_controller.py:137-158` returns a hardcoded fake token `'fake-jwt-token-' + str(user.id)` that is never validated by any middleware. No route checks for auth tokens or session validity.
Impact: Any anonymous user can delete all data, change user roles to admin, create arbitrary users, and access all reports. The login endpoint gives a false impression of security.
Recommendation: Implement JWT-based authentication. Add an auth middleware that validates tokens on all endpoints except login. Use Flask-JWT-Extended or similar. Remove the fake token from the login response.

### [CRITICAL] Hardcoded Credentials / Secrets
File: config.py:8
Description: `SECRET_KEY = os.environ.get('SECRET_KEY', 'dev-secret-key-change-in-production')` — the fallback value `'dev-secret-key-change-in-production'` is a known string committed to source control. Flask uses this key for session cookie signing.
Impact: If deployed without setting the SECRET_KEY env var, an attacker can forge session cookies and impersonate any user. The fallback value is predictable and publicly visible in the repository.
Recommendation: Remove the default fallback or raise an error at startup if SECRET_KEY is not set in production. Example: `SECRET_KEY = os.environ.get('SECRET_KEY') or (_raise_or_warn('SECRET_KEY not set'))`.

### [HIGH] N+1 Query Problem
File: controllers/category_controller.py:14, controllers/report_controller.py:23-24, controllers/report_controller.py:49-50, controllers/report_controller.py:97-105
Description: Multiple instances of queries inside loops:
- `category_controller.py:14` — `Task.query.filter_by(category_id=cat.id).count()` inside list comprehension iterating all categories. N categories = N queries.
- `report_controller.py:23-24` — `for p in range(1, 6): priorities[p] = Task.query.filter_by(priority=p).count()`. 5 iterations = 5 queries instead of one GROUP BY.
- `report_controller.py:49-50` — `for user in users: user_tasks = Task.query.filter_by(user_id=user.id).all()`. N users = N queries.
- `report_controller.py:97-105` — Loads all tasks with `.all()` then iterates in Python to count statuses, instead of using `db.func.count()` with GROUP BY.
Impact: Performance degrades linearly with data volume. A report with 100 users triggers 100+ database queries. The summary endpoint is especially vulnerable.
Recommendation: Replace loop-based queries with aggregate SQL queries using `db.func.count()` and `group_by()`. For category task counts, use a single query with GROUP BY. For report stats, use a single query returning counts by status/priority.

### [MEDIUM] Inadequate Error Handling
File: controllers/*.py (all controllers)
Description: All controllers return error tuples `({'error': 'message'}, status_code)` directly instead of raising the `AppError` exception defined in `middlewares/__init__.py:7-11`. The centralized error handler infrastructure (`register_error_handlers`) exists but is never exercised by normal controller flow. Example at `controllers/task_controller.py:27`: `return {'error': 'Task não encontrada'}, 404` instead of `raise AppError('Task não encontrada', 404)`.
Impact: Error responses are inconsistent — some come from the middleware (500, 404 from Flask), others from manual tuples in controllers. The `AppError` class and its handler are dead code. Adding cross-cutting concerns like logging or error metrics requires modifying every controller method.
Recommendation: Have controllers raise `AppError` for expected errors (not found, validation). Let the middleware handle formatting and logging. Keep unexpected errors caught by the generic 500 handler.

### [MEDIUM] Magic Numbers / Magic Strings
File: controllers/task_controller.py:205-208, controllers/report_controller.py:17-19, controllers/report_controller.py:75-79
Description: Status strings are hardcoded directly in multiple locations despite `VALID_STATUSES` constant existing in `utils/helpers.py:22`:
- `task_controller.py:205-208` — `'pending'`, `'in_progress'`, `'done'`, `'cancelled'` used directly in `stats()` method instead of referencing `VALID_STATUSES`.
- `report_controller.py:17-19` — Same status strings repeated.
- `report_controller.py:75-79` — Priority labels (`'critical'`, `'high'`, `'medium'`, `'low'`, `'minimal'`) mapped to numbers 1-5 without constants.
- `report_controller.py:23` — `range(1, 6)` is a magic range for priority levels.
Impact: If a status name changes, it must be updated in multiple files. The VALID_STATUSES constant exists but is not consistently used, creating a maintenance gap.
Recommendation: Use the existing `VALID_STATUSES` and `DEFAULT_PRIORITY` constants consistently. Define a priority-to-label mapping constant (e.g., `PRIORITY_LABELS = {1: 'critical', ...}`) in `utils/helpers.py` and reference it everywhere.

### [MEDIUM] Unused Dependencies
File: requirements.txt:4-5
Description: `marshmallow==3.20.1` and `requests==2.31.0` are declared in `requirements.txt` but never imported or used anywhere in the codebase. The project uses manual validation in controllers and makes no HTTP requests to external services.
Impact: Unnecessary dependencies increase container image size, broaden the attack surface, and confuse developers who might think marshmallow schemas are being used for validation.
Recommendation: Remove `marshmallow` and `requests` from `requirements.txt`. If marshmallow validation is planned, add it back when implementing.

### [LOW] Unused Code
File: utils/helpers.py:15-19, services/notification_service.py:30-47
Description: `parse_date()` function at `utils/helpers.py:15-19` is defined but never called anywhere in the codebase. `NotificationService.notify_task_assigned()` and `NotificationService.notify_task_overdue()` (lines 30-47) are defined but no controller calls them — notifications are never triggered.
Impact: Dead code increases cognitive load and may mislead developers into thinking notifications are active.
Recommendation: Either wire up the notification service to task controllers (call `notify_task_assigned` on task creation/update) or remove the unused methods and the `parse_date` function.

### [LOW] Debug Artifacts in Production Code
File: seed.py:21-23, app.py:13
Description: `seed.py:21-23` uses extremely weak passwords (`'1234'`, `'abcd'`, `'pass'`) for seed users. While seed data, these could accidentally end up in production if seed is run against a prod database. `app.py:13` sets `logging.basicConfig(level=logging.INFO)` globally at module level, which cannot be adjusted per-environment.
Impact: Seed passwords could be exploitable if the database is not re-seeded with proper credentials. Global logging config may be too verbose or too quiet depending on environment.
Recommendation: Use clearly fake passwords in seed like `'SEED_PASSWORD_CHANGE_ME'` or generate random ones and print them. Move logging configuration to `Config` class to allow per-environment control.

================================
Total: 8 findings
================================
