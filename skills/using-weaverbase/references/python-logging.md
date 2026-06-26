# Python Logging

Use the bundled logging configuration as the default template for Python apps unless the user or project explicitly specifies a different logging preference.

Bundled template: `references/python-logging/log-config.json`

## Default rule

- Copy or apply `references/python-logging/log-config.json` into the user's app repository before referencing it from app code, startup commands, Dockerfiles, or deployment config.
- Reference the copied app-local config at runtime. Do not make the app depend on the skill file path because skill files are usually outside the app repository and build context.
- Apply the app-local config during app startup so the format is active before application code emits logs.
- Use the standard `logging` module for application logs. Do not use `print()` for application logging.
- Keep user- or project-specified logging requirements when they exist, but otherwise prefer this shared config over inventing a new format.

## Recommended app-local location

Prefer a stable path that is included in source control and container build context, such as `config/log-config.json`.

When creating or updating an app, copy the bundled template contents into that app-local file first, then reference the app-local path:

```python
import json
import logging.config
from pathlib import Path

LOG_CONFIG_PATH = Path("config/log-config.json")

logging.config.dictConfig(json.loads(LOG_CONFIG_PATH.read_text(encoding="utf-8")))
```

After configuration, create loggers with `logging.getLogger(__name__)` and log through that logger.

```python
import logging

logger = logging.getLogger(__name__)

logger.info("Application started")
```
