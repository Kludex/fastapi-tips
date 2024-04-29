# 101 FastAPI Tips by The FastAPI Expert

This repository contains trips and tricks for FastAPI. If you have any tip that you believe is useful, feel free
to open an issue or a pull request.

Consider sponsor me on GitHub to support my work. With your support, I will be able to create more content like this.

[![GitHub Sponsors](https://img.shields.io/badge/Sponsor%20me%20on-GitHub-%23EA4AAA)](https://github.com/sponsors/Kludex)

## 1. Install `uvloop` and `httptools`

By default, [Uvicorn][uvicorn] doesn't comes with `uvloop` and `httptools` which are faster than the default
asyncio event loop and HTTP parser. You can install them using the following command:

```bash
pip install uvloop httptools
```

[Uvicorn][uvicorn] will automatically use them if they are installed in your environment.

## 2. Be careful with non-async functions

There's a performance penalty when you use non-async functions in FastAPI. So, always prefer to use async functions.
The penalty comes from the fact that FastAPI will call [`run_in_threadpool`][run_in_threadpool], which will run the
function using a thread pool.

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
from httpx import AsyncClient


async def main():
    async with AsyncClient(app=app, base_url="http://test") as client:
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
from httpx import AsyncClient
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
    async with LifespanManager(app, lifespan) as manager:
        async with AsyncClient(app=manager.app) as client:
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
        await client.aclose()


app = FastAPI(lifespan=lifespan)


@app.get("/")
async def read_root(request: Request):
    client = request.app.state.client
    response = await client.get("/")
    return response.json()
```

Using the lifespan state, you'd do something like this:

```py
from contextlib import asynccontextmanager
from typing import Any, AsyncIterator, TypedDict, cast

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

## 7. Use a main router to group all sub-routers instead of using the `app` directly

Instead of using the `app` directly, you can use a main router to group all sub-routers.

```py
# project/routers/__init__.py
import fastapi

from project.routers.account import account
from project.routers.admin import admin

main = fastapi.APIRouter()

main.include_router(account)
main.include_router(admin)
```

```py
# project/app.py
import project.routers


app = fastapi.FastAPI(
    title="API",
    description="API for project",
    version="0.1.0",
)

app.include_router(project.routers.main)

# And many other configurations...
```

This way, you can keep your `app` file short and clean. The configuration of the app 
will be in the `app` file, and the management of the routers will be in the 
`routers` package.

This is really useful when you have feature flag logic that include a router or not. 
This code won't bloat the `app` file and will be easier to maintain.

[uvicorn]: https://www.uvicorn.org/
[run_sync]: https://anyio.readthedocs.io/en/stable/threads.html#running-a-function-in-a-worker-thread
[run_in_threadpool]: https://github.com/encode/starlette/blob/9f16bf5c25e126200701f6e04330864f4a91a898/starlette/concurrency.py#L36-L42
[increase-threadpool]: https://anyio.readthedocs.io/en/stable/threads.html#adjusting-the-default-maximum-worker-thread-count
[websockets-iter-data]: https://www.starlette.io/websockets/#iterating-data
[florimondmanca]: https://github.com/sponsors/florimondmanca
[asgi-lifespan]: https://github.com/florimondmanca/asgi-lifespan
[lifespan state]: https://asgi.readthedocs.io/en/latest/specs/lifespan.html#lifespan-state
