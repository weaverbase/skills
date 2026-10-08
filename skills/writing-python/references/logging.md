# Python Logging

Use the bundled logging configuration as the default template for Python apps unless the user or project explicitly specifies a different logging preference.

Bundled template: [logging/log-config.json](logging/log-config.json)

Colors are off by default (`use_colors: false`) because logs are usually collected from containers, where ANSI escape codes corrupt aggregated output; enable them only in a local developer copy.

## Default Rules

- Copy or apply the bundled `log-config.json` into the user's app repository before referencing it from app code, startup commands, Dockerfiles, or deployment config.
- Reference the copied app-local config at runtime. Do not make the app depend on the skill file path, because skill files are usually outside the app repository and build context.
- Apply the app-local config during app startup so the format is active before application code emits logs.
- Use the standard `logging` module with appropriate log levels for application logs. Do not use `print()` for application logging.
- Keep secrets, tokens, passwords, and personal data out of logs, including exception details and request payloads.
- Keep user- or project-specified logging requirements when they exist, but otherwise prefer this shared config over inventing a new format.

For FastAPI/Uvicorn apps, also pass the app-local config to Uvicorn at startup; see [fastapi.md](fastapi.md).

The bundled access-log format includes client addresses and raw request lines, which can expose personal data or query-string credentials. Keep the template as a configuration starting point, not a redaction guarantee: disable raw access logs (`--no-access-log` or `access_log=False` in Uvicorn) until the app has a verified sanitizing filter or equivalent safe access-log pipeline. Redact sensitive exception and application-log fields too; changing the format alone does not sanitize them.

## Recommended App-Local Location

Prefer a stable path that is included in source control and the container build context, such as `config/log-config.json`.

The bundled formatter factories depend on `uvicorn.logging`. In an app that already uses Uvicorn, copy the template into the app-local file and load that path. Do not add Uvicorn just for logging in a standard-library-only app: apply this equivalent standard-library variant to the copied configuration before startup instead.

```python
import json
import logging.config
from pathlib import Path

LOG_CONFIG_PATH = Path("config/log-config.json")

config = json.loads(LOG_CONFIG_PATH.read_text(encoding="utf-8"))
# Standard-library variant for an app that does not use Uvicorn.
config["formatters"] = {
    "default": {"format": "%(asctime)s %(levelname)s %(message)s"},
}
config["handlers"].pop("access", None)
config["loggers"].pop("uvicorn.access", None)
config["loggers"].pop("uvicorn.error", None)
logging.config.dictConfig(config)
```

After configuration, create loggers with `logging.getLogger(__name__)` and log through that logger.

```python
import logging

logger = logging.getLogger(__name__)

logger.info("Application started")
```

## No print() for Application Logging

Good:

```python
logger.info("Order processing completed")
```

Bad:

```python
print("Order processing completed")
```

## Common Mistakes

- Using `print()` logging in application code.
- Skipping the bundled template for a Python app when the user or project has not specified another logging preference.
- Referencing the bundled template from app runtime code, Dockerfiles, or deployment commands instead of copying it into the app repository first.
- Starting the app (or FastAPI/Uvicorn) before applying the app-local copied logging config.
- Inventing a new log format when no other preference was specified.
