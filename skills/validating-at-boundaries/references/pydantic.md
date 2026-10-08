# Pydantic v2

How to apply the rules in [../SKILL.md](../SKILL.md) in Python with Pydantic v2. Models own shape and validation; application operations receive validated models.

The `EmailStr` examples require Pydantic's email-validation extra (`pydantic[email]`); the settings example uses `pydantic-settings`. Use the repository's dependency policy before adding either.

## Model Configuration

Use `model_config` with `ConfigDict` on `BaseModel` classes, or `SettingsConfigDict` on `BaseSettings` classes. Do not create nested `Config` classes; they are Pydantic v1 style.

Bad: nested `Config` class.

```python
class MyModel(BaseModel):
    name: str

    class Config:
        frozen = True
```

## Shared Base, Create/Update Variants, Response Model

Define each concept's constraints once. Subclass a shared base for create payloads, reuse constrained field types for optional update fields, and return an explicit response model so internal fields (password hashes, internal IDs) cannot leak.

```python
from typing import Annotated

from pydantic import BaseModel, ConfigDict, EmailStr, Field


UserName = Annotated[str, Field(min_length=1, max_length=100)]


class UserBase(BaseModel):
    email: EmailStr
    name: UserName


class CreateUser(UserBase):
    model_config = ConfigDict(extra="forbid", frozen=True)


class UpdateUser(BaseModel):
    model_config = ConfigDict(extra="forbid", frozen=True)

    email: EmailStr | None = None
    name: UserName | None = None


class UserOut(UserBase):
    model_config = ConfigDict(from_attributes=True)

    id: int
```

- `UpdateUser` redeclares fields as optional because Pydantic has no built-in `partial()`. Reuse constrained aliases such as `UserName` so constraints stay identical rather than duplicating them.
- `extra="forbid"` rejects unknown keys on inbound request bodies. Leave the default (extra keys ignored) for models that parse external API responses, so new upstream fields do not break you.
- `UserOut` uses `from_attributes=True` so an ORM row maps to a typed model (for example `UserOut.model_validate(row)`) in the persistence code. Operations then use that typed value without re-parsing it.

## Applying an Update

Do not use `model_copy(update=...)` to apply untrusted data. It copies an instance without running validation. Validate the update payload first. If its fields have weaker constraints than the destination model, validate the newly assembled value too:

```python
updated = UserBase.model_validate(
    user.model_dump() | patch.model_dump(exclude_unset=True)
)
```

Here `patch` is an already validated `UpdateUser`. Its optional fields allow explicit `None`, but `UserBase` does not, so blindly copying a patch could create an invalid instance. Validate the assembled value before treating it as `UserBase`, and translate any validation failure at the boundary. This checks a new shape, not the original request again. `model_copy(update=...)` is suitable only when the update is trusted and already guarantees the destination constraints. Decide explicitly whether the API treats null as clearing a field or as invalid.

## Strict Mode

Avoid `strict=True` on models that parse query params, form data, or environment variables. In Python mode, strict mode rejects coercions such as `"1"` to `1` and string to `datetime`.

## Settings

Parse configuration once at startup with a typed settings model. Do not hardcode configuration values, and do not read environment variables ad hoc inside operations.

```python
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="APP_", env_file=".env")

    database_url: str
    api_key: SecretStr
    request_timeout_s: float = Field(default=10.0, gt=0)
```

## Error Ownership

The boundary owns validation errors. The operation owns domain errors and never raises validation errors.

```python
from fastapi import HTTPException


class UserAlreadyExists(Exception):
    pass


async def create_user(data: CreateUser) -> User:
    if await users.exists_by_email(data.email):
        raise UserAlreadyExists(data.email)
    ...


@router.post("/users", response_model=UserOut, status_code=201)
async def post_user(body: CreateUser) -> User:
    try:
        return await create_user(body)
    except UserAlreadyExists:
        raise HTTPException(status_code=409, detail="Email already registered")
```

FastAPI turns a `ValidationError` on `body` into a 422 response before the operation runs. Outside a framework that does this for you, catch `pydantic.ValidationError` at the boundary and translate it to a 400/422 response or a form error there.

Branching on legitimately optional or empty values, such as `if user.nickname is None` or `if not order.items`, is domain logic and stays in the operation. Do not repeat constraints the model already guarantees (for example `len(name) > 100` when `name` has `max_length=100`).

## Bad: The Operation Re-Validates Raw Input

```python
def create_user(data: dict) -> User:
    if "@" not in data.get("email", ""):
        raise ValueError("bad email")
    if len(data.get("name", "")) > 100:
        raise ValueError("name too long")
    ...
```

This accepts a raw `dict`, repeats checks the model should own, and raises validation errors from inside the operation. Define the model, validate at the boundary, and accept `CreateUser`.
