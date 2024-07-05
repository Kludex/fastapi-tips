# 101 FastAPI Tips by [The FastAPI Expert]

This repository contains trips and tricks for FastAPI. If you have any tip that you believe is useful, feel free
to open an issue or a pull request.

Consider sponsor me on GitHub to support my work. With your support, I will be able to create more content like this.

[![GitHub Sponsors](https://img.shields.io/badge/Sponsor%20me%20on-GitHub-%23EA4AAA)](https://github.com/sponsors/Kludex)

> [!TIP]
    Remember to **watch this repository** to receive notifications about new tips.

## 1. Install `uvloop` and `httptools`

By default, [Uvicorn][uvicorn] doesn't comes with `uvloop` and `httptools` which are faster than the default
asyncio event loop and HTTP parser. You can install them using the following command:

```bash
pip install uvloop httptools
```

[Uvicorn][uvicorn] will automatically use them if they are installed in your environment.

> [!WARNING]
> `uvloop` can't be installed on Windows. If you use Windows locally, but Linux on production, you can use
> an [environment marker](https://peps.python.org/pep-0496/) to not install `uvloop` on Windows
> e.g. `uvloop; sys_platform != 'win32'`.

## 2. Be careful with non-async functions

You should only use non-async functions (`def`) when you have either:
1. Thread blocking IO e.g.:
    ```py
    import requests
    from fastapi import FastAPI

    app = FastAPI()


    @app.get("/")
    def get_request() -> None:
        requests.get("https://example.org/")
    ```
2. CPU bound blocking that escapes the GIL e.g.:
   ```py
   ```

When you use a non-async function, FastAPI will call [`run_in_threadpool`][run_in_threadpool], which will run the
function using a thread pool. If your code isn't running one of the above two points, always prefer to use an async
function.

> [!NOTE]
    Internally, [`run_in_threadpool`][run_in_threadpool] will use [`anyio.to_thread.run_sync`][run_sync] to run the
    function in a thread pool.

> [!TIP]
    There are only 40 threads available in the thread pool. If you use all of them, your application will be blocked.

    To change the number of threads available, you can use the following code:

```py
import anyio
from contextlib import asynccontextmanager
from typing import Iterator

from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI) -> Iterator[None]:
    limiter = anyio.to_thread.current_default_thread_limiter()
    limiter.total_tokens = 100
    yield

app = FastAPI(lifespan=lifespan)
```

You can read more about it on [AnyIO's documentation][increase-threadpool].

## 3. Use `async for` instead of `while True` on WebSocket

Most of the examples you will find on the internet use `while True` to read messages from the WebSocket.

I believe the uglier notation is used mainly because the Starlette documentation didn't show the `async for` notation for a long time.

Instead of using the `while True`:

```py
from fastapi import FastAPI
from starlette.websockets import WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket) -> None:
    await websocket.accept()
    while True:
        data = await websocket.receive_text()
        await websocket.send_text(f"Message text was: {data}")
```

You can use the `async for` notation:

```py
from fastapi import FastAPI
from starlette.websockets import WebSocket

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket) -> None:
    await websocket.accept()
    async for data in websocket.iter_text():
        await websocket.send_text(f"Message text was: {data}")
```

You can read more about it on the [Starlette documentation][websockets-iter-data].

## 4. Ignore the `WebSocketDisconnect` exception

If you are using the `while True` notation, you will need to catch the `WebSocketDisconnect`.
The `async for` notation will catch it for you.

```py
from fastapi import FastAPI
from starlette.websockets import WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket) -> None:
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Message text was: {data}")
    except WebSocketDisconnect:
        pass
```

If you need to release resources when the WebSocket is disconnected, you can use that exception to do it.

If you are using an older FastAPI version, only the `receive` methods will raise the `WebSocketDisconnect` exception.
The `send` methods will not raise it. In the latest versions, all methods will raise it.
In that case, you'll need to add the `send` methods inside the `try` block.

## 5. Use HTTPX's `AsyncClient` instead of `TestClient`

Since you are using `async` functions in your application, it will be easier to use HTTPX's `AsyncClient`
instead of Starlette's `TestClient`.

```py
from fastapi import FastAPI


app = FastAPI()


@app.get("/")
async def read_root():
    return {"Hello": "World"}


# Using TestClient
from starlette.testclient import TestClient

client = TestClient(app)
response = client.get("/")
assert response.status_code == 200
assert response.json() == {"Hello": "World"}

# Using AsyncClient
import anyio
from httpx import AsyncClient, ASGITransport


async def main():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        response = await client.get("/")
        assert response.status_code == 200
        assert response.json() == {"Hello": "World"}


anyio.run(main)
```

If you are using lifespan events (`on_startup`, `on_shutdown` or the `lifespan` parameter), you can use the
[`asgi-lifespan`][asgi-lifespan] package to run those events.

```py
from contextlib import asynccontextmanager
from typing import AsyncIterator

import anyio
from asgi_lifespan import LifespanManager
from httpx import AsyncClient, ASGITransport
from fastapi import FastAPI


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    print("Starting app")
    yield
    print("Stopping app")


app = FastAPI(lifespan=lifespan)


@app.get("/")
async def read_root():
    return {"Hello": "World"}


async def main():
    async with LifespanManager(app) as manager:
        async with AsyncClient(transport=ASGITransport(app=manager.app)) as client:
            response = await client.get("/")
            assert response.status_code == 200
            assert response.json() == {"Hello": "World"}


anyio.run(main)
```

> [!NOTE]
    Consider supporting the creator of [`asgi-lifespan`][asgi-lifespan] [Florimond Manca][florimondmanca] via GitHub Sponsors.

## 6. Use Lifespan State instead of `app.state`

Since not long ago, FastAPI supports the [lifespan state], which defines a standard way to manage objects that need to be created at
startup, and need to be used in the request-response cycle.

The `app.state` is not recommended to be used anymore. You should use the [lifespan state] instead.

Using the `app.state`, you'd do something like this:

```py
from contextlib import asynccontextmanager
from typing import AsyncIterator

from fastapi import FastAPI, Request
from httpx import AsyncClient


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    async with AsyncClient(app=app) as client:
        app.state.client = client
        yield


app = FastAPI(lifespan=lifespan)


@app.get("/")
async def read_root(request: Request):
    client = request.app.state.client
    response = await client.get("/")
    return response.json()
```

Using the lifespan state, you'd do something like this:

```py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from typing import Any, TypedDict, cast

from fastapi import FastAPI, Request
from httpx import AsyncClient


class State(TypedDict):
    client: AsyncClient


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[State]:
    async with AsyncClient(app=app) as client:
        yield {"client": client}


app = FastAPI(lifespan=lifespan)


@app.get("/")
async def read_root(request: Request) -> dict[str, Any]:
    client = cast(AsyncClient, request.state.client)
    response = await client.get("/")
    return response.json()
```

## 7. Enable AsyncIO debug mode

If you want to find the endpoints that are blocking the event loop, you can enable the AsyncIO debug mode.

When you enable it, Python will print a warning message when a task takes more than 100ms to execute.

Run the following code with `PYTHONASYNCIODEBUG=1 python main.py`:

```py
import os
import time

import uvicorn
from fastapi import FastAPI


app = FastAPI()


@app.get("/")
async def read_root():
    time.sleep(1)  # Blocking call
    return {"Hello": "World"}


if __name__ == "__main__":
    uvicorn.run(app, loop="uvloop")
```

If you call the endpoint, you will see the following message:

```bash
INFO:     Started server process [19319]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO:     127.0.0.1:50036 - "GET / HTTP/1.1" 200 OK
Executing <Task finished name='Task-3' coro=<RequestResponseCycle.run_asgi() done, defined at /uvicorn/uvicorn/protocols/http/httptools_impl.py:408> result=None created at /uvicorn/uvicorn/protocols/http/httptools_impl.py:291> took 1.009 seconds
```

You can read more about it on the [official documentation](https://docs.python.org/3/library/asyncio-dev.html#debug-mode).

## 8. Implement a Pure ASGI Middleware instead of `BaseHTTPMiddleware`

The [`BaseHTTPMiddleware`][base-http-middleware] is the simplest way to create a middleware in FastAPI.

> [!NOTE]
> The `@app.middleware("http")` decorator is a wrapper around the `BaseHTTPMiddleware`.

There were some issues with the `BaseHTTPMiddleware`, but most of the issues were fixed in the latest versions.
That said, there's still a performance penalty when using it.

To avoid the performance penalty, you can implement a [Pure ASGI middleware]. The downside is that it's more complex to implement.

Check the Starlette's documentation to learn how to implement a [Pure ASGI middleware].

## 9. Your dependencies may be running on threads

If the function is non-async and you use it as a dependency, it will run in a thread.

In the following example, the `http_client` function will run in a thread:

```py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from httpx import AsyncClient
from fastapi import FastAPI, Request, Depends


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[dict[str, AsyncClient]]:
    async with AsyncClient() as client:
        yield {"client": client}


app = FastAPI(lifespan=lifespan)


def http_client(request: Request) -> AsyncClient:
    return request.state.client


@app.get("/")
async def read_root(client: AsyncClient = Depends(http_client)):
    return await client.get("/")
```

To run in the event loop, you need to make the function async:
```py
# ...

async def http_client(request: Request) -> AsyncClient:
    return request.state.client

# ...
```

As an exercise for the reader, let's learn a bit more about how to check the running threads.

You can run the following with `python main.py`:

```py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

import anyio
from anyio.to_thread import current_default_thread_limiter
from httpx import AsyncClient
from fastapi import FastAPI, Request, Depends


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[dict[str, AsyncClient]]:
    async with AsyncClient() as client:
        yield {"client": client}


app = FastAPI(lifespan=lifespan)


# Change this function to be async, and rerun this application.
def http_client(request: Request) -> AsyncClient:
    return request.state.client


@app.get("/")
async def read_root(client: AsyncClient = Depends(http_client)): ...


async def monitor_thread_limiter():
    limiter = current_default_thread_limiter()
    threads_in_use = limiter.borrowed_tokens
    while True:
        if threads_in_use != limiter.borrowed_tokens:
            print(f"Threads in use: {limiter.borrowed_tokens}")
            threads_in_use = limiter.borrowed_tokens
        await anyio.sleep(0)


if __name__ == "__main__":
    import uvicorn

    config = uvicorn.Config(app="main:app")
    server = uvicorn.Server(config)

    async def main():
        async with anyio.create_task_group() as tg:
            tg.start_soon(monitor_thread_limiter)
            await server.serve()

    anyio.run(main)
```

If you call the endpoint, you will see the following message:

```bash
❯ python main.py
INFO:     Started server process [23966]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
Threads in use: 1
INFO:     127.0.0.1:57848 - "GET / HTTP/1.1" 200 OK
Threads in use: 0
```

Replace the `def http_client` with `async def http_client` and rerun the application.
You will not see the message `Threads in use: 1`, because the function is running in the event loop.

> [!TIP]
> You can use the [FastAPI Dependency] package that I've built to make it explicit when a dependency should run in a thread.

[uvicorn]: https://www.uvicorn.org/
[run_sync]: https://anyio.readthedocs.io/en/stable/threads.html#running-a-function-in-a-worker-thread
[run_in_threadpool]: https://github.com/encode/starlette/blob/9f16bf5c25e126200701f6e04330864f4a91a898/starlette/concurrency.py#L36-L42
[increase-threadpool]: https://anyio.readthedocs.io/en/stable/threads.html#adjusting-the-default-maximum-worker-thread-count
[websockets-iter-data]: https://www.starlette.io/websockets/#iterating-data
[florimondmanca]: https://github.com/sponsors/florimondmanca
[asgi-lifespan]: https://github.com/florimondmanca/asgi-lifespan
[lifespan state]: https://asgi.readthedocs.io/en/latest/specs/lifespan.html#lifespan-state
[The FastAPI Expert]: https://github.com/Kludex
[base-http-middleware]: https://www.starlette.io/middleware/#basehttpmiddleware
[pure ASGI middleware]: https://www.starlette.io/middleware/#pure-asgi-middleware
[FastAPI Dependency]: https://github.com/kludex/fastapi-dependency
