# JobRadar fork notes

This fork runs only freehire's collection engine for JobRadar. Upstream: https://github.com/strelov1/freehire (MIT).
Branch `jobradar` carries the fork's own changes; `main` tracks upstream unchanged.

## What is used
- `cmd/harvest-ats resolve` — company website -> careers page -> ATS board (internal/ingest/atsdetect)
- `cmd/harvest-boards --apply` — probes each candidate board on the platform's API, keeps live ones (`boards` table)
- `cmd/ingest <provider>` — crawls the provider's boards, writes postings to `jobs`
- `cmd/migrate` — schema

Not used: web/API server, auth, billing, mail, auto-apply, search (Meilisearch), Redis, S3.

## Local setup (macOS)
    brew install go postgresql@17 pgvector
    export LC_ALL=en_US.UTF-8
    initdb -D ~/Documents/Repos/freehire-pgdata -U jobradar --auth=trust -E UTF8
    pg_ctl -D ~/Documents/Repos/freehire-pgdata -o "-p 5433 -k /tmp" -l ~/Documents/Repos/freehire-pgdata/pg.log start
    createdb -h localhost -p 5433 -U jobradar freehire
    # Apply migrations one file per session, as docker-entrypoint-initdb.d does (0001 clears search_path,
    # which breaks cmd/migrate on an empty database), then record them:
    for f in migrations/*.sql; do psql "$DATABASE_URL" -q -v ON_ERROR_STOP=1 -f $f; done
    go build -o bin/ ./cmd/migrate ./cmd/ingest ./cmd/harvest-boards ./cmd/harvest-ats ./cmd/add-board
    DATABASE_URL=postgres://jobradar@localhost:5433/freehire?sslmode=disable ./bin/migrate -baseline

The JobRadar side (index, matching, salary threshold, coverage, report) lives in
~/Documents/Repos/job-radar: `freehire_run.sh` and `freehire_bridge.py`.

## Updating from upstream
    git fetch upstream
    git checkout main && git merge --ff-only upstream/main
    git checkout jobradar && git merge main
    go build -o bin/ ./cmd/... (the five commands above)
    DATABASE_URL=... ./bin/migrate        # new migrations only
