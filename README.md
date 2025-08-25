# FastAPI Error Map: Per-Endpoint Error Handling & OpenAPI

[![Releases](https://img.shields.io/badge/Releases-Download-blue?logo=github)](https://github.com/sandeepveeramachaneni/fastapi-error-map/releases)

Fast, clear error handling for FastAPI endpoints that keeps OpenAPI in sync. Map exceptions per endpoint, document response schemas in OpenAPI, and generate accurate Swagger UI docs.

---

[![PyPI](https://img.shields.io/pypi/v/fastapi-error-map?logo=python)](https://pypi.org/project/fastapi-error-map)  [![License](https://img.shields.io/github/license/sandeepveeramachaneni/fastapi-error-map)](https://github.com/sandeepveeramachaneni/fastapi-error-map/blob/main/LICENSE)  [![Python Versions](https://img.shields.io/pypi/pyversions/fastapi-error-map)](https://pypi.org/project/fastapi-error-map)

Images
- FastAPI: ![FastAPI logo](https://fastapi.tiangolo.com/img/logo-margin/logo-teal.png)
- Swagger UI: ![Swagger logo](https://petstore.swagger.io/images/petstore-logos.png)
- Python: ![Python logo](https://www.python.org/static/community_logos/python-logo.png)

Topics: api, apiroute, apirouter, error-handling, errors, exception-handling, exceptions, fastapi, openapi, python, swagger-ui

Releases: You must download and execute the release file for your platform from https://github.com/sandeepveeramachaneni/fastapi-error-map/releases. The release page contains packaged files and installers. Choose the file for your platform, download it, and run the provided executable or script. The release link appears above and again in the Releases section below.

---

## Why this library

Developers often handle errors in one central place. That works for server logic. It fails for per-endpoint behavior. Some endpoints must return different error shapes. Some must document multiple status codes. FastAPI lets you add `responses` to endpoints. You can also raise exceptions. But keeping code and OpenAPI synced is manual work.

This library solves that. You map errors on a per-endpoint basis. The library generates OpenAPI entries for each mapped error. It keeps docs up to date. It preserves FastAPI performance. It uses APIRoute and APIRouter hooks so it fits with existing patterns.

Benefits:
- Map errors to response models and status codes per endpoint.
- Auto-add response models to OpenAPI.
- Keep Swagger UI accurate and helpful.
- Use simple code to declare error maps.
- Work with FastAPI dependencies and background tasks.
- Test error shapes easily.

This README gives deep, clear examples. It includes design notes, usage patterns, tests, CI tips, and migration help.

---

## Quick concepts

- Error map: a dictionary that maps exception types to an error response schema and HTTP status code.
- Response model: a Pydantic model for the error body.
- APIRoute subclass: the class that intercepts endpoints to attach error mapping and update OpenAPI.
- Responses in OpenAPI: the part of the schema that lists possible status codes and content schemas.

Core idea: annotate an endpoint with an error map. The route class wraps the endpoint handler. It catches mapped exceptions. It returns the mapped response. It registers response schemas in the OpenAPI schema. The Swagger UI shows correct errors and example payloads.

---

## Install

Install from PyPI:

```bash
pip install fastapi-error-map
```

Or add to your project dependencies:

```bash
pip install fastapi fastapi-error-map[extra]
```

If you prefer releases, download and execute the release file for your platform from:
https://github.com/sandeepveeramachaneni/fastapi-error-map/releases

Pick the asset for your OS. Run the installer or script included in the release. The release includes prebuilt wheels, tarballs, and optional CLI tools.

---

## Basic usage

This example shows a minimal app that maps errors for a single endpoint.

app.py

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from fastapi_error_map import ErrorMapRouter, ErrorMapRoute
from fastapi_error_map.types import ErrorDescriptor

app = FastAPI()
router = ErrorMapRouter(route_class=ErrorMapRoute)


class NotFoundError(Exception):
    pass


class ConflictError(Exception):
    pass


class ErrorResponse(BaseModel):
    code: str
    message: str


@router.get("/items/{item_id}", response_model=dict, responses={}, tags=["items"])
async def get_item(item_id: int):
    if item_id == 1:
        return {"id": 1, "name": "Item One"}
    if item_id == 2:
        raise NotFoundError()
    raise ConflictError()


# Attach error map on the router level or per-route via decorator
item_error_map = {
    NotFoundError: ErrorDescriptor(status_code=404, model=ErrorResponse),
    ConflictError: ErrorDescriptor(status_code=409, model=ErrorResponse),
}

router.add_error_map("/items/{item_id}", "GET", item_error_map)

app.include_router(router)
```

This code:
- Creates a route class that uses error maps.
- Declares two custom exceptions.
- Declares an error model.
- Attaches an error map for the endpoint.
- The library updates OpenAPI with 404 and 409 responses and the error model.

In Swagger UI, the endpoint shows possible error responses and the model for the response body.

---

## Advanced integration

You can use a decorator to attach error maps directly to endpoint functions. This keeps mapping next to the handler.

```python
from fastapi import FastAPI
from pydantic import BaseModel
from fastapi_error_map import error_map

app = FastAPI()


class BadRequest(Exception):
    pass


class ErrorOut(BaseModel):
    detail: str
    hint: str | None = None


@error_map({
    BadRequest: {"status_code": 400, "model": ErrorOut}
})
@app.get("/search")
async def search(q: str):
    if not q:
        raise BadRequest()
    return {"results": []}
```

The decorator registers the error map with the route. It also updates the OpenAPI schema to include the 400 response for the `/search` endpoint.

---

## ErrorDescriptor and schema

The library exposes a small helper type. It keeps code clear and explicit.

Example:

```python
from pydantic import BaseModel
from fastapi_error_map.types import ErrorDescriptor

class ErrorModel(BaseModel):
    code: str
    message: str

err = ErrorDescriptor(status_code=422, model=ErrorModel, description="Validation failed")
```

Fields:
- status_code: HTTP status code.
- model: Pydantic model or reference to one.
- description: optional string that shows in OpenAPI.
- example: optional example payload.

You can pass a plain dict instead of ErrorDescriptor for quick use:

```python
{
    MyError: {"status_code": 401, "model": ErrorModel}
}
```

The library normalizes that to ErrorDescriptor.

---

## Per-endpoint, per-router, global maps

You can attach error maps in three places:

1. Per endpoint — precise control.
2. Per router — default for routes in the router.
3. Global — fallback for any unmapped handlers.

Priority:
- Endpoint map overrides router map.
- Router map overrides global map.

Examples:

```python
# Global map
app.state.error_map = {
    GlobalError: ErrorDescriptor(500, GlobalErrorOut)
}

# Router map
router = ErrorMapRouter(error_map={
    RouterError: ErrorDescriptor(400, RouterErrorOut)
})

# Endpoint-level map
@router.get("/foo")
@error_map({SpecificError: {"status_code": 409, "model": SpecificErrorOut}})
async def foo():
    ...
```

---

## Automatic OpenAPI sync

FastAPI builds OpenAPI from route metadata. This library changes the route metadata before FastAPI builds OpenAPI. It sets the `responses` argument with correct schema references. It also sets `response_model` for error responses where appropriate.

The library:
- Registers Pydantic models in the component schemas section.
- Adds response content types (application/json) with example bodies.
- Attaches descriptions and headers when provided.

The net result: Swagger UI reflects the mappings the same as runtime behavior.

---

## Implementation notes

Core components:
- ErrorMapRoute: APIRoute subclass that wraps the endpoint handler.
- ErrorMapRouter: APIRouter subclass that uses ErrorMapRoute by default and exposes helpers.
- error_map decorator: syntactic sugar to attach maps to endpoints.
- ErrorDescriptor: small DTO to normalize input.

Error handling flow:
1. The route wrapper executes the endpoint handler.
2. The wrapper catches exceptions.
3. If the exception type appears in the endpoint map, the wrapper formats a response using the provided model and status code.
4. If the exception is not mapped, the wrapper either re-raises it or invokes a fallback handler.
5. The wrapper returns a Response with the mapped status code and JSON body produced by the model.

OpenAPI flow:
1. On mount or when building OpenAPI, the router iterates routes.
2. For each route, the router fetches the effective error map.
3. The router merges error descriptors into the route's `responses` attribute.
4. FastAPI consumes that data to build the final OpenAPI schema.

---

## Examples: error shapes and examples

Define an error model with an example:

```python
from pydantic import BaseModel, Field

class ErrorModel(BaseModel):
    code: str = Field(..., example="ITEM_NOT_FOUND")
    message: str = Field(..., example="The item with id 2 does not exist")
    details: dict | None = Field(None, example={"missing_id": 2})
```

Use this in a descriptor:

```python
map = {
    ItemNotFound: ErrorDescriptor(
        status_code=404,
        model=ErrorModel,
        description="Item not found",
        example={"code": "ITEM_NOT_FOUND", "message": "Missing item", "details": {"missing_id": 2}}
    )
}
```

Swagger UI will show the example for 404 responses.

---

## Patterns and best practices

- Keep error models small. Use `code` and `message` fields.
- Use enums for error codes when you maintain a fixed set of errors.
- Include optional `details` only when needed.
- Use one top-level error model per API surface if you need consistent structure.
- Use different models for different endpoints if the error payload needs endpoint-specific fields.
- Map HTTP status codes carefully. Use 4xx for client issues and 5xx for server issues.
- If you use custom middleware for exceptions, ensure it does not intercept mapped exceptions before ErrorMapRoute.

---

## Integrating with FastAPI dependencies

Dependency code may raise exceptions. Map those exceptions as well. ErrorMapRoute can intercept them if the exception bubbles out of the dependency.

Example:

```python
from fastapi import Depends, HTTPException

def get_current_user(token: str = Depends(oauth2_scheme)):
    if not valid(token):
        raise UnauthorizedError()

@error_map({UnauthorizedError: {"status_code": 401, "model": ErrorOut}})
@app.get("/secret")
async def secret(user = Depends(get_current_user)):
    return {"ok": True}
```

When `get_current_user` raises `UnauthorizedError`, the route returns the mapped 401 response.

---

## OpenAPI examples and multiple media types

By default the library registers application/json for response content. If you need other media types, provide them explicitly in the descriptor.

```python
ErrorDescriptor(
    status_code=415,
    model=ErrorModel,
    content={"application/problem+json": {"schema": {"$ref": "#/components/schemas/ErrorModel"}}}
)
```

The library uses the provided `content` as-is if present.

---

## Testing

Write tests for mapped behavior and for OpenAPI documentation.

Unit test for handler behavior:

```python
from fastapi.testclient import TestClient
from app import app

client = TestClient(app)

def test_not_found():
    resp = client.get("/items/2")
    assert resp.status_code == 404
    assert "code" in resp.json()
```

Test OpenAPI contains mapped responses:

```python
def test_openapi_contains_error():
    schema = client.get("/openapi.json").json()
    responses = schema["paths"]["/items/{item_id}"]["get"]["responses"]
    assert "404" in responses
    assert "409" in responses
    assert responses["404"]["content"]["application/json"]["schema"]["$ref"].endswith("ErrorResponse")
```

Testing patterns:
- Use TestClient for runtime assertions.
- Validate OpenAPI JSON for accurate docs.
- Use snapshot tests when you expect no changes.

---

## Migration guide

If you have code that uses global exception handlers:

- Identify endpoints that return special error payloads.
- Create Pydantic models for those payloads.
- Add endpoint-level descriptors or router-level maps.
- Remove or keep global handlers for unmapped exceptions.
- Run the test suite. Validate OpenAPI JSON.

Steps:

1. Install the package.
2. Create error models.
3. Add ErrorMapRoute or ErrorMapRouter.
4. Attach maps.
5. Validate docs and runtime behavior.

---

## Performance

The library uses fast path logic:

- It only wraps endpoints that include mappings or belong to routers that opt in.
- It registers error schemas during route mounting, not per request.
- Error handling path uses simple type checks and Pydantic model serialization.

You should not see measurable overhead in normal request handling. Error paths still pay Pydantic serialization cost.

---

## Deployment tips

- Build wheels for your CI pipeline.
- Ensure the package version aligns with your lock file.
- When releasing, include wheels and source tarball in GitHub releases.
- The release page contains downloadable files. Download and execute the release file for your platform from: https://github.com/sandeepveeramachaneni/fastapi-error-map/releases

The releases page includes:
- Prebuilt wheels for Python versions supported.
- Source tarballs.
- Optional CLI binaries for simple project setup.

---

## CLI utilities

The release bundles include a small CLI tool to scan your project and generate default error maps. Use it to bootstrap mappings.

Example usage (after downloading the release file and running it):

```bash
fastapi-error-map init --project src/ --out error_maps.py
```

The CLI uses static analysis and common heuristics to suggest mappings. Review generated code before use.

---

## Contributing

Guidelines:
- Create issues for feature requests and bugs.
- Open a pull request for fixes or new features.
- Write tests for new behavior.
- Keep changes small and focused.
- Respect existing API shapes and backward compatibility.
- Use black and isort for formatting.

Local dev:
1. Clone the repo.
2. Create a virtualenv.
3. Install dev deps:
   ```bash
   pip install -e ".[dev]"
   ```
4. Run tests:
   ```bash
   pytest
   ```

When you prepare a release:
- Update CHANGELOG.md.
- Bump version in pyproject.toml.
- Build distributions:
  ```bash
  python -m build
  ```
- Upload to PyPI or attach builds to GitHub release.

The GitHub release page is the central point for downloads. Download and execute the release file for your platform from: https://github.com/sandeepveeramachaneni/fastapi-error-map/releases

---

## API reference

Key exports:

- ErrorMapRouter: APIRouter subclass. Use `route_class` to customize.
- ErrorMapRoute: APIRoute subclass. Wrap endpoint handlers.
- error_map: decorator to attach error maps.
- ErrorDescriptor: type to specify status_code, model, description, example.
- utils: functions to merge maps and render response bodies.

Example signatures:

```python
class ErrorDescriptor:
    status_code: int
    model: Type[BaseModel] | None
    description: str | None
    example: dict | None
    content: dict | None
```

Decorator:

```python
def error_map(mapping: Dict[Type[Exception], Union[ErrorDescriptor, dict]]):
    ...
```

Router helper:

```python
class ErrorMapRouter(APIRouter):
    def __init__(self, *args, error_map: dict | None = None, **kwargs):
        ...
    def add_error_map(self, path: str, method: str, mapping: dict):
        ...
```

Route class:

```python
class ErrorMapRoute(APIRoute):
    def __init__(self, *args, error_map: dict | None = None, **kwargs):
        ...
```

---

## Real-world examples

Example 1: Multi-tenant API with per-tenant error code mappings.

- Use a router per tenant with different error models.
- Attach router-level maps to reflect tenant-specific business rules.

Example 2: Public API with a global error model and endpoint-specific details.

- Use a global error model `ApiError`.
- Add endpoint maps for endpoints that need specialized fields in `details`.

Example 3: Legacy service migration.

- Wrap old code with FastAPI endpoints.
- Create error maps to present consistent JSON to new clients.
- Use OpenAPI to communicate the new contract.

---

## FAQ

Q: How does this interact with FastAPI's global exception handlers?
A: ErrorMapRoute handles exceptions at the route level. If an exception is mapped, the route handles it before global handlers. If not mapped, the exception can bubble to a global handler.

Q: Can I use this with background tasks?
A: Yes. The mapping only applies to exceptions raised during the request handler execution. Background tasks run after the response. Use logging and background error reporting for those.

Q: Can I map built-in exceptions like ValueError?
A: Yes. Map any exception type. Use caution mapping very broad types unless you intend to catch them for many endpoints.

Q: Does this support asynchronous exceptions from dependencies?
A: Yes. Exceptions that bubble to the endpoint are part of the mapping.

---

## Security

Handle error content carefully. Do not include sensitive data in error messages. Keep error models generic in public APIs. Use codes and hints that avoid revealing internal state.

---

## Roadmap

Planned features:
- OpenAPI extension points for richer examples.
- CLI improvements for project scanning.
- Support for mapping non-JSON media types easily.
- Built-in common error models for quick use.
- More tests and sample projects.

---

## Changelog

Check the Releases page for binary assets and release notes. Download and execute the release file appropriate for your environment from:
https://github.com/sandeepveeramachaneni/fastapi-error-map/releases

Release notes include:
- Breaking changes.
- Migration steps.
- Binary assets.

---

## License

This project uses the license file included in the repository. Check LICENSE in the repo root for full terms.

---

Files and examples

- examples/basic.py — Minimal working example.
- examples/decorator.py — Shows error_map decorator.
- examples/cli-init.md — How to use the init CLI.
- docs/ — Additional docs and design notes.

Images and badges used in this README come from public resources and are for documentation.

---

Maintain concise models, write tests, and keep OpenAPI and code in sync.