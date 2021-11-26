# Error Handling

The error handling in APIFlask is based on the following basic concepts:

- All the automatic errors (404, 405, 500) will be in JSON format by default.
- Use `abort` and `HTTPError` to generate an error response.
- Use `app.errror_processor` (`app` is an instance of `apiflask.APIFlask`) to register a
  custom error response processor.

!!! tip

    The error handler registered with `app.errorhandler` for specific HTTP errors will be
    used over the custom error response processor.


## Automatic JSON error response

In Flask, for 400/404/405/500 errors, a default error response will be generated.
The default error response will be in HTML format with a default error message and error
description. However, in APIFlask, these errors will be returned in JSON format with
the following preset fields:

- `message`: The HTTP reason phrase or a custom error description.
- `detail`: An empty dict (404/405/500) or the error details of the request validation (400).

You can control this behavior with the `json_errors` parameter when creating the APIFlask
instance, and it defaults to `True`:

```python
from apiflask import APIFlask

# this will disable the automatic JSON error response
app = APIFlask(__name__, json_errors=False)
```

You can use the `app.error_processor` decorator to register a custom error processor
to customize the error response body. See more details
[here](/error-handling/#custom-error-response-processor).


## Make an error response with `abort` and `HTTPError`

There are two ways to abort the request handling process and return an error response
in the view function:

- Call the `abort` function

Just like what you do in a normal Flask view function, but this `abort` function is
provided by APIFlask:

```python
from apiflask import abort


@app.route('/')
def hello():
    abort(400, message='Something is wrong...')
    return 'Hello, world!'  # this line will never be reached
```

It will raise an `HTTPError` behind the scene, so it will take the same arguments (see below).

- Raise the `HTTPError` class

Raise `HTTPError` will do the same thing:

```python
from apiflask import HTTPError


@app.route('/')
def hello():
    raise HTTPError(400, message='Something is wrong...')
    return 'Hello, world!'  # this line will never be reached
```

The call will generate an error response like this:

```json
{
    "message": "Something is wrong...",
    "detail": {}
}
```

Here are all the parameters you can pass to `abort` and `HTTPError`:

| Name | Type | Default Value | Description |
| ---- | ---- | ------------- | ----------- |
| `status_code` | int | - | The status code of the error response. |
| `message` | string | - | The error message. Used as the `message` field in the response body. |
| `detail` | dict | `{}` | The detailed information of the validation error. Used as the `detail` field in the response body. |
| `headers` | dict | - | The headers of the error response. |
| `extra_data` | dict | - | Additional fields to be added to the body of the error response body. |

The `extra_data` is useful when you want to add more fields to the response body, for example:

```python
abort(
    400,
    message='Something is wrong...',
    extra_data={
        'docs': 'http://example.com',
        'error_code': 1234
    }
)
```
will produce the below response:

```json
{
    "message": "Something is wrong...",
    "detail": {},
    "docs": "http://example.com",
    "error_code": 1234
}
```


## Custom error status code and description

The following configuration variables can be used to customize the validation and
authentication errors:

- `VALIDATION_ERROR_DESCRIPTION`
- `AUTH_ERROR_DESCRIPTION`
- `VALIDATION_ERROR_STATUS_CODE`
- `AUTH_ERROR_STATUS_CODE`

See the [Response customization](/configuration#response-customization) section in the
configuration docs for the details.


## Custom error response processor

You can use the `app.error_processor` decorator to register a custom error response
processor function. It's a global error processor for all HTTP errors.

The decorated callback function will be called in the following situations:

- Any HTTP exception is raised by Flask when `APIFlask(json_errors=True)` (default).
- A validation error happened when parsing a request.
- An exception triggered with [`HTTPError`][apiflask.exceptions.HTTPError]
- An exception triggered with [`abort`][apiflask.exceptions.abort].

If you have set the `json_errors` argument to `True` when creating the `app`
instance, this callback function will also be used for normal HTTP errors,
for example, 404 and 500 errors, etc. You can still register a specific error
handler for a specific error code or exception with the
`app.errorhandler(code_or_exection)` decorator. In that case, the return
value of the error handler will be used as the response when the corresponding
error or exception happens.

The callback function must accept an error object as an argument and return a valid
response:

```python
from apiflask import APIFlask

app = APIFlask(__name__)


@app.error_processor
def my_error_processor(error):
    return {
        'status_code': error.status_code,
        'message': error.message,
        'detail': error.detail
    }, error.status_code, error.headers
```

The error object is an instance of [`HTTPError`][apiflask.exceptions.HTTPError],
so you can get error information via its attributes:

- status_code: If the error is triggered by a validation error, the value will be
    400 (default) or the value you passed in config `VALIDATION_ERROR_STATUS_CODE`.
    If the error is triggered by [`HTTPError`][apiflask.exceptions.HTTPError]
    or [`abort`][apiflask.exceptions.abort], it will be the status code
    you passed. Otherwise, it will be the status code set by Werkzueg when
    processing the request.
- message: The error description for this error, either you passed or grabbed from
    Werkzeug.
- detail: The detail of the error. When the validation error happens, it will
    be filled automatically in the following structure:

    ```python
    "<location>": {
        "<field_name>": ["<error_message>", ...],
        "<field_name>": ["<error_message>", ...],
        ...
    },
    "<location>": {
        ...
    },
    ...
    ```

    The value of `location` can be `json` (i.e., request body) or `query`
    (i.e., query string) depending on the place where the validation error
    happened.
- headers: The value will be `{}` unless you pass it in `HTTPError` or `abort`.
- extra_data: Additional error information passed with `HTTPError` or `abort`.

If you want, you can rewrite the whole response body to anything you like:

```python
@app.error_processor
def my_error_processor(error):
    body = {
        'error_message': error.message,
        'error_detail': error.detail,
        'status_code': error.status_code
    }
    return body, error.status_code, error.headers
```

!!! tip

    I would recommend keeping the `detail` in the response since it contains
    the detailed information about the validation error when it happened.

After you change the error response, you have to update the corresponding OpenAPI schema
for error responses so the API docs will match your custom error response schema.


## Update the OpenAPI schema of error responses

There are two error schemas in APIFlask: one for generic errors (including auth errors),
and one for validation errors. They can be configured with `HTTP_ERROR_SCHEMA` and
`VALIDATION_ERROR_SCHEMA`, respectively.

!!! question "Why do we need two schemas for error responses?"

    The reason behind a separate schema for the validation error response is that the `detail`
    field of the validation errors will always have values. While for generic HTTP errors,
    the `detail` field will be empty unless you passed something with `HTTPError` and
    `abort`.

When you change the error response body with `error_processor`, you will also need
to update the error response schema, so it will update the OpenAPI spec of the error
response. The schema can be a dict of OpenAPI schema or a marshmallow schema class.
Here is an example that adds a `status_code` field to the default error response
and renames the existing fields (with OpenAPI schema dict):

```python
# use the built-in `validation_error_detail_schema` for the `detail` field
from apiflask import APIFlask
from apiflask.schemas import validation_error_detail_schema


# schema for generic error response, including auth errors
http_error_schema = {
    "properties": {
        "error_detail": {
            "type": "object"
        },
        "error_message": {
            "type": "string"
        },
        "status_code": {
            "type": "integer"
        }
    },
    "type": "object"
}


# schema for validation error response
validation_error_schema = {
    "properties": {
        "error_detail": validation_error_detail_schema,
        "error_message": {
            "type": "string"
        },
        "status_code": {
            "type": "integer"
        }
    },
    "type": "object"
}

app = APIFlask(__name__)
app.config['VALIDATION_ERROR_SCHEMA'] = validation_error_schema
app.config['HTTP_ERROR_SCHEMA'] = http_error_schema
```


## Handling authentication errors

When you set the `json_errors` to `True` when creating the APIFlask instance (defaults to `True`),
APIFlask will return JSON errors for auth errors and use the built-in errors callback or the
error processor you created with `app.error_processor`.

In the following situations, you need to register a separate error processor for auth
errors:

- If you want to make some additional process for 401/403 error, instead of using
  `app.errorhandler(401)` or `app.errorhandler(403)` to register a specific error
  handler, you have to use `auth.error_processor` to register an auth error processor.
- If you have set `json_errors` to `False`, but also want to customize the error
  response, you also need to register a custom auth error processor since the global
  error processor will not be used.

You can use the `auth.error_processor` decorator to register an auth error processor. It
works just like `app.error_processor`:

```python
from apiflask import HTTPTokenAuth

auth = HTTPTokenAuth()


@auth.error_processor
def my_auth_error_processor(error):
    body = {
        'error_message': error.message,
        'error_detail': error.detail,
        'status_code': error.status_code
    }
    return body, error.status_code, error.headers
```

If you registered an auth error processor when `json_error` is `True`, it will overwrite the
global error processor.

!!! question "Why do we need a separate error processor for auth errors?"

    APIFlask's authentication feature is backed with Flask-HTTPAuth. Since Flask-HTTPAuth
    uses a separate error handler for its errors, APIFlask has to add a separate
    error processor to handle it. We may figure out a simple way for this in the future.