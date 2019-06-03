# Runtime

A runtime represents a standalone isolated server that exposes APIs for remote code execution and output. The following document specifies a common standard that must be adopted by runtimes.

## Port

The runtime must always run on internal port `1111`. The communication protocol will be HTTP.

## Healthcheck

The runtime must expose a `/ping` route that will be used for checking the health of the runtime. Response must include a 200 status code.

## Configuration

The server will pass in the following configuration variables as environment variables:

```
  # The base URL of the notebook backend
  UNKLEARN_SERVER_URI
  
  # Redis broker URL for socket conn
  UNKLEARN_REDIS_BROKER_URL
```

## Socket communication

The runtime must send the output of code or file execution via `socketio` using Redis broker. If there is no socketio implementation for the runtime's language, then use the following server route:

```
  POST UNKLEARN_SERVER_URI/api/v1/cells/results
  {
    "sid": ...the session id, passed in for execution,
    "cellId": ..id of the cell,
    "notebookId": ..id of the notebook,
    "error": ...stderr,
    "output": ...output
  }
```

Otherwise, if a broker is available then emit `cell_result` event with the following json payload:

```
{
  "cellId": ..id of the cell,
  "notebookId": ..id of the notebook,
  "error": ...stderr,
  "output": ...output
}
# Send it to the room with id `sid`
room=sid

# Use socket namespace `/cells`
namespace=/cells

```


## REPL mode
