# dask-notes

## Unconfirmed

### Port in use error when running a `dask.distributed` client from a script

The following has given me a port in use forever looping error:

```python
from dask.distributed import Client

with Client() as client:
    client.compute(delayed(my_func)(), sync=True)
```

Embedding in a `__main__` check resolves:

```python
from dask.distributed import Client

if __name__ == '__main__':
    with Client() as client:
        client.compute(delayed(my_func)(), sync=True)
```

## Mistakes I've made

### `delayed` objects

Calling `delayed(func).compute()` for `func` that doesn't take any parameters instead of `delayed(func)().compute()`.
