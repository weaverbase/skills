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
