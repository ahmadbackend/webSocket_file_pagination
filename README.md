# WebSocket File Pagination

Streams a large text file to the browser in fixed-size chunks over a WebSocket, so the client
can page through it without loading the whole file into memory or re-reading it from the start
on every request.

Built with Django Channels on ASGI.

## How it works

The server holds the read position for the lifetime of the connection. Each message the client
sends is a request for the next chunk:

```
client  ──▶  {"message": "next"}
server  ──▶  {"chunk": ["line 1", "line 2", ... 100 lines]}
```

`FileRead` (in [`WebSock/consumers.py`](WebSock/consumers.py)) is an `AsyncWebsocketConsumer`
that keeps a `file_position` offset per connection. On each request it reopens the file with
`aiofiles`, seeks to the stored offset, reads the next 100 lines, records the new offset with
`tell()`, and returns the batch. Reaching EOF yields a short chunk and then empty ones.

Because reads go through `aiofiles`, the event loop is never blocked while the disk is busy —
one slow read does not stall the other connections on the worker.

### Why offsets rather than line numbers

Seeking to a byte offset is O(1). Tracking line numbers would mean re-reading the file from the
beginning on every request to count newlines, which turns paging through an N-chunk file into
O(N²) work.

## Project layout

```
ReadFile/        Django project — settings, ASGI entrypoint
WebSock/         the app
  consumers.py   FileRead — the websocket consumer
  routing.py     websocket URL routing
  templates/     demo client page
```

## Running it

```bash
git clone https://github.com/ahmadbackend/webSocket_file_pagination.git
cd webSocket_file_pagination

python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt

cp .env.example .env             # then set FILE_PATH and DJANGO_SECRET_KEY
python manage.py migrate
python manage.py runserver
```

Django Channels installs Daphne, so `runserver` serves ASGI and the websocket endpoint works
without a separate process.

## Configuration

| Variable | Purpose |
|---|---|
| `DJANGO_SECRET_KEY` | Required. Generate with `django.core.management.utils.get_random_secret_key()` |
| `DJANGO_DEBUG` | `True` in development; defaults to `False` |
| `DJANGO_ALLOWED_HOSTS` | Comma-separated hostnames |
| `FILE_PATH` | Path to the file to paginate |

Chunk size is currently fixed at 100 lines in `read_next_chunk()`.
