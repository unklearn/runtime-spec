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

Using HTTP incurs extra payload + parsing time, so use socket broker if it is available and only use the HTTP mechanism as a fallback.

## Modes

A runtime may support one or more of the following modes:

* `Interactive` - REPL like interactive code execution
* `File` - Code execution via files mounted in the runtime container
* `Endpoint` - Dynamic routes exposed by container to execute custom methods
* `Daemon` - Support long running processes with a proxy that port-forwards for client requests.

The supported modes must be declared by the runtime at startup, so that the backend can relay this information to the client.
For example, if a runtime does not support file mode, then a cell that operates within that runtime will not have the file selector enabled.

For detailed information see sections below:


## Interactive mode

The runtime may support an interactive code execution mode similar to Jupyter notebook cell execution. Code will be distributed amongst cells, and execution order is determined by client.

The interactive mode may support `shell` language in addition to the host language if necessary, e.g `python`, `javascript` etc.

Endpoints: 
* GET `/interactive?code={str}&channel={str}&cellId={str}&language={str}
* POST `/interactive?language={str}`
  ```
    {
      "code": str,
      "channel": str,
      "cellId": str
    }
  ```

`shell` execution is asnychronous, but `language` specific execution may be synchrounous or asynchronous. To help the front-end understand when code execution is finished, use the following events:

### Before execution/failure
```
  Event: 'cell_run_start'
  Args: channel, notebookId, cellId, status: 'busy'
```

### During execution (success or error)
```
  Event: 'cell_log',
  Args: channel, notebookId, cellId, output: [...str of stdout], error: [...stderr if any, else empty]
```

### After execution/failure
```
  Event: 'cell_run_end',
  Args: channel, notebookId, cellId, status: 'done' or 'error'
```


