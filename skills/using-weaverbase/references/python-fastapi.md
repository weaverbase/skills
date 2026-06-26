# Python and FastAPI

Use modern Python syntax and async-safe FastAPI patterns by default.

## Python General

### Type hints

Use Python 3.10+ union syntax with `|`. Avoid `typing.Union` and `typing.Optional`.

Good:

```python
def process(value: str | None) -> dict[str, int | float]:
    ...
```

Bad:

```python
from typing import Dict, Optional, Union


def process(value: Optional[str]) -> Dict[str, Union[int, float]]:
    ...
```

### Dictionary merging

Use the dict merge operator. Avoid unpacking merges and mutation when creating a merged value.

Good:

```python
merged = defaults | overrides
```

Bad:

```python
merged = {**defaults, **overrides}
defaults.update(overrides)
```

### Python PDF libraries

When working with PDFs in Python, use `pypdfium2`. Do not use `pdf2image` or PyMuPDF (`fitz`).

Good:

```python
import pypdfium2 as pdfium
```

Bad:

```python
from pdf2image import convert_from_path
import fitz
```

## FastAPI / Modern Python Web

### Uvicorn logging

Use `references/python-logging/log-config.json` as the default Uvicorn logging template unless the user or project specifies another logging preference. Copy or apply that template into the user's app repository first, then pass the app-local config path when Uvicorn starts so the application and access log format is available before the app emits startup logs.

Do not point Uvicorn at `references/python-logging/log-config.json` at runtime. Skill files are usually outside the app repository and container build context.

Recommended app-local path: `config/log-config.json`

CLI startup:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --log-config config/log-config.json
```

Programmatic startup:

```python
from pathlib import Path

import uvicorn

from app.main import app

LOG_CONFIG_PATH = Path("config/log-config.json")

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000, log_config=str(LOG_CONFIG_PATH))
```

### Async database operations

Use async database drivers and async ORM methods in async applications. Avoid synchronous drivers that block the event loop.

Good examples:

- PostgreSQL: `asyncpg`
- MongoDB: `motor`
- MySQL/MariaDB: `asyncmy`
- SQLAlchemy: async sessions and async queries

### Dependency injection

Use `Annotated` with `Depends` for FastAPI dependencies. Avoid `Depends()` directly as a parameter default.

Good:

```python
from typing import Annotated

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession


async def get_db() -> AsyncSession:
    ...


async def endpoint(db: Annotated[AsyncSession, Depends(get_db)]):
    ...
```

Bad:

```python
async def endpoint(db: AsyncSession = Depends(get_db)):
    ...
```
