# F1. “I want scheduled file-sync”

## Value

Scheduled sync keeps your server responsive while new files are indexed at predictable times. Fresh metadata appears automatically—no manual triggers required.

---

## Minimal `docker-compose.yml`

```yaml
services:
  home-index:
    image: ghcr.io/nashspence/home-index:latest
    environment:
      - CRON_EXPRESSION=* * * * *        # every minute; edit to taste
      - METADATA_DIRECTORY=/home-index/metadata
    volumes:
      - ./input:/files:ro                # place source files here (read-only)
      - ./output:/home-index             # logs, metadata, appear here
    depends_on:
      - meilisearch

  meilisearch:
    image: getmeili/meilisearch:latest
    volumes:
      - ./output/meili:/meili_data       # search index persists on host
```

---

## User Testing

```bash
# 0. (Optional) change cadence:
#    edit CRON_EXPRESSION in docker-compose.yml, e.g.
#    - CRON_EXPRESSION=*/5 * * * *      # every five minutes

mkdir -p input output                    # create bind-mount folders

# 1. Ensure there is at least ONE file to index
echo "hello world" > input/hello.txt     # example seed file

# 2. Launch the stack
docker compose up -d                     # pull images & start services

# 3. Watch it run
tail -f output/files.log                 # see a new “start file sync” line each tick
```

**What you’ll see after the first tick**

```
output/
├ files.log
├ metadata/
│   └ by-id/
│       ├ <xxhash-of-hello.txt>/
│       │   ├ document.json
│       │   └ …                       ← any side-car artifacts
│       └ …                           ← one folder per file hash in ./input
└ meili/
    └ …                               ← search index files
```

Change the cron schedule any time by editing the compose file and running:

```bash
docker compose up -d --force-recreate
```

The new cadence takes effect immediately; the rhythm in **`output/files.log`** updates accordingly.

---

## Input ↔ Output

| **Your action**                       | **Observable result**                                                                                                                                                        |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Add / modify a file in `./input/`** | On the next tick a **directory named by that file’s xxhash** appears under **`./output/metadata/by-id/`** containing `document.json` and any other artifacts for that file. |
| **Look at `./output/files.log`**      | One line per cron tick—e.g. `2025-06-22 14:00:00,012 start file sync`—spaced exactly as defined by `CRON_EXPRESSION`.                                                        |
| **Run `docker compose stop`**         | Current tick finishes, a final log entry is written, containers halt; everything in **`./output/`** remains.                                                                 |
| **Run `docker compose rm -fsv`**      | Containers & named volumes are removed, but the bind-mounted **`./output/`** directory stays on disk for inspection or backup.                                               |

---

## Acceptance

1. **Cadence fidelity** Time between any two consecutive “start file sync” lines is **never shorter** than the cron interval you configured (verified ±1 s).
2. **Clean slate per start** `./output/` is wiped and rebuilt on every container start, so logs & metadata always correspond to *that* run.
3. **Proof of successful indexing** With at least one file in `./input/`, the first tick creates a directory under `./output/metadata/by-id/`, and MeiliSearch data persists in `./output/meili/`.
