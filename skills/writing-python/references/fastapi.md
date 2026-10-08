# FastAPI

Use async-safe FastAPI patterns by default.

## Uvicorn Logging

Use [logging/log-config.json](logging/log-config.json) as the default Uvicorn logging template unless the user or project specifies another logging preference. Copy or apply that template into the user's app repository first, then pass the app-local config path when Uvicorn starts so the configured format is available before the app emits startup logs. Disable raw access logs until a verified sanitizing pipeline removes client addresses and sensitive request data; the template itself does not redact them.

Do not point Uvicorn at the skill's `log-config.json` at runtime. Skill files are usually outside the app repository and container build context.

Recommended app-local path: `config/log-config.json`

CLI startup:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --log-config config/log-config.json --no-access-log
```

Programmatic startup:

```python
from pathlib import Path

import uvicorn

LOG_CONFIG_PATH = Path("config/log-config.json")

if __name__ == "__main__":
    uvicorn.run("app.main:app", host="0.0.0.0", port=8000, log_config=str(LOG_CONFIG_PATH), access_log=False)
```

Pass the import string rather than importing `app.main` above this call, so Uvicorn applies the logging configuration before application import-time logs are emitted.

See [logging.md](logging.md) for the general logging rules and for configuring logging in non-Uvicorn Python apps.

## Async Database Operations

Use async database drivers and async ORM methods in async applications. Avoid synchronous drivers that block the event loop.

Good examples:

- PostgreSQL: `asyncpg`
- MongoDB: `pymongo` async API (`AsyncMongoClient`); `motor` is deprecated
- MySQL/MariaDB: `asyncmy`
- SQLAlchemy: async sessions and async queries

## Dependency Injection

Use `Annotated` with `Depends` for FastAPI dependencies. Avoid `Depends()` directly as a parameter default.

Good:

```python
from collections.abc import AsyncIterator
from typing import Annotated

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker

SessionLocal: async_sessionmaker[AsyncSession]  # created at startup


async def get_db() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:
        yield session


async def endpoint(db: Annotated[AsyncSession, Depends(get_db)]):
    ...
```

Bad:

```python
async def endpoint(db: AsyncSession = Depends(get_db)):
    ...
```

## Common Mistakes

- Using `Depends()` as a parameter default instead of `Annotated[..., Depends(...)]`.
- Using a synchronous database driver or blocking ORM call inside an async endpoint or worker.
- Starting Uvicorn without the app-local copied log config, then fixing logging after startup.
- Pointing the Uvicorn `--log-config` option or `log_config=` argument at the skill's file path instead of the app-local copy.
