When working on a project:

- If a `compose.yaml` or `docker-compose.yml` exists, run commands via `docker compose exec` / `docker compose run --rm` rather than on the host (php, composer, symfony, npm, psql, redis-cli, etc.). Fall back to host only when Docker is unavailable or the user explicitly asks.
- When adding a new dependency (database, cache, queue, search engine, mail catcher...) or scaffolding a new project, default to a Compose service rather than installing on the host.
- If the project shows Docker signals (`Dockerfile`, `.dockerignore`) but no Compose file, ask before suggesting host-level commands.
- Use `docker compose` (v2 plugin syntax), not the legacy `docker-compose` binary, unless the project pins the old one.
