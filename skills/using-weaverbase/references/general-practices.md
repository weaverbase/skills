# General Best Practices

Use validated configuration, appropriate logging, and modern test frameworks.

## Configuration

Use environment variables with validation. Avoid hardcoded configuration values.

Good examples:

- Python: Pydantic Settings with typed fields and constraints.
- TypeScript: Zod schema for `process.env` at startup or boundary initialization.

## Logging

Use the standard logging module with appropriate log levels. Avoid `print()` statements for application logging.

Good:

```python
logger.info("User registered: %s", user.id)
```

Bad:

```python
print(f"user registered: {user.id}")
```

## Testing

Use modern test frameworks:

- Python: `pytest`
- JavaScript/TypeScript: `vitest` or `jest`

Avoid `unittest` for new Python projects unless maintaining an existing `unittest` suite or a project constraint requires it.
