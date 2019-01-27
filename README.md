# dask-notes

## High performance computing (cluster) environments

Try out [Dask-Jobqueue](https://github.com/dask/dask-jobqueue) for computing your graphs on a cluster with minimal code modification.

See [this post](http://www.ericmjl.com/blog/2018/10/11/parallel-processing-with-dask-on-gridengine-clusters/) by [@ericmjl](https://github.com/ericmjl) for a tutorial on how to use this package.

## Working with Dask bags

### Iterating over items

Items passed to your function from a bag can only be iterated over once (they are `itertools.chain` objects). You would have to `list(items)` them to iterate over multiple times.

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
