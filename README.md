# Broken-auth Laravel lab (local only)

**Current state: VULNERABLE.** Phase 1 is live; the fixes are *not* applied.
Phase 2 (hardening) is planned only — see `ATTACK.md`.

Nothing here binds outside loopback: the app listens on `127.0.0.1:8086` only.
No Docker, no containers, no services, no restart policy.

## Run

```bash
cd broken-auth-lab
./start.sh                 # http://127.0.0.1:8086/login
curl -i http://127.0.0.1:8086/login
python3 exploit.py         # prints both flags
./reset.sh                 # restore user@lab.local to role=user
./stop.sh                  # stop the PHP server
```

Demo accounts:

| Email | Password | Seeded role |
|---|---|---|
| `user@lab.local` | `Password123!` | `user` |
| `admin@lab.local` | `AdminPass123!` | `admin` |

The bundled PHP 8.3 runtime in `runtime/` is used for every Artisan command and
for the built-in server; the framework router is
`app/vendor/laravel/framework/src/Illuminate/Foundation/resources/server.php`.
`start.sh` writes `runtime/php.ini` and sets `PHP_INI_SCAN_DIR`,
`LD_LIBRARY_PATH`, `SESSION_DRIVER=file`, and the PID file `.server.pid`.

## What is broken

1. `/admin` checks only that a session exists — no Gate, Policy, or role check —
   so any logged-in user reads `FLAG{broken-auth-missing-gate}`.
2. `POST /profile` calls `User::find(session('user_id'))->update($request->all())`
   with `role` in the model's `$fillable` and no FormRequest validation, so a
   normal user can POST `role=admin` and unlock
   `FLAG{broken-auth-mass-assignment}` on `/dashboard`.
   CSRF is exempted for `login` and `profile` so plain curl/Python clients work.

Full vulnerable snippets, walkthrough, impact and Phase 2 fix preview are in
`ATTACK.md`.

## File layout

| Path | Purpose |
|---|---|
| `start.sh` / `stop.sh` | Start/stop the loopback-only PHP server (PID file `.server.pid`) |
| `reset.sh` | `php artisan migrate:fresh --seed --force` via bundled PHP |
| `exploit.py` | Python 3 stdlib exploit (CookieJar session, both flags) |
| `ATTACK.md` | Vulnerable code, exploit steps, impact, Phase 2 fixes |
| `runtime/` | Bundled PHP 8.3 binary, libs, extensions, generated `php.ini` |
| `app/app/Http/Controllers/AuthController.php` | Hand-rolled login/dashboard/logout |
| `app/app/Http/Controllers/AdminController.php` | Missing role check (vuln A) |
| `app/app/Http/Controllers/ProfileController.php` | Mass assignment (vuln B) |
| `app/app/Models/User.php` | `$fillable` includes `role` |
| `app/database/migrations/*_add_role_to_users_table.php` | Adds `users.role` |
| `app/database/seeders/UserSeeder.php` | Exactly two seeded accounts |
| `app/resources/views/{login,dashboard,admin}.blade.php` | Lab UI and flags |
| `app/bootstrap/app.php` | CSRF exemptions for `login` and `profile` |
| `app/.env` | SQLite + `SESSION_DRIVER=file`, `APP_URL=http://127.0.0.1:8086` |

## AGENTS.md note

The generated Laravel app's `AGENTS.md` conventions were followed: controllers,
migration, seeder, and views were generated with `php artisan make:*`; relevant
rules under `.agents/skills/` were reviewed (there is no `.ai/rules` directory);
and `vendor/bin/pint --format agent` was run after PHP edits. The missing
authorization, the deliberately broad `$fillable`, and the CSRF exemptions are
**intentional, documented deviations** from the security best practices — they
are the lab target and are not to be fixed in Phase 1.
