Overview of Dask graphs
-------------------------------------

See official docs: http://docs.dask.org/en/latest/high-level-graphs.html

Under the hood, objects like Dask `Bag`s, `DataFrame`s, and `Array`s are
high-level tools used to generate a low-level computational graph - a series of
operations (mapping, reducing, arithmetic, ...) applied to data. The low-level
specification of the graph that these containers spit out are then what is used
by the Dask scheduler itself to perform the number crunching.

Dask graphs are fundamentally dictionaries where the keys are identifiers of
data entities, generally strings or tuples of a string + a number (at least if
the graph has been generated using Dask collections). Entities in this case
refer to either inputs to the graph or outputs of a graph operation. A
dictionary value is then how that particular entity is computed: the list of
operations necessary to calculate this one "unit" of data.

Generally, a data entity's generation "formula" is designed such that it
contains as few operations as possible; the graph is kept very fine-grained so
as to promote as much graph optimization as can be achieved. Complex
computations are therefore handled through establishing the dependencies of data
entities to others as defined in the graph dictionary.

Dependencies in the graph - what previously calculated data entities a
particular entity depends on - can be determined by crawling through the
dictionary's values for a given entity. Dictionary values themselves, if the
corresponding data entity depends on a previous one, will include the data
entity keys corresponding to its dependencies to indicate that those data
variables are necessary inputs to a particular graph node's calculations.

Given a Dask graph (not dictionary) generated through a series of Dask
collection commands (e.g., `graph = df.sum().std()...`), you can extract the
actual `dict` object (our low-level representation) by calling:
`graph.__dask_graph__()`.


Deeper dive into graph specification
-------------------------------------

While a user has a bit of leeway in, for example, the implementation of the
graph dictionary for graphs they generate by hand, graph dictionaries created by
operations on Dask collections seem to abide by the following rules (some rules
are universal regardless of whether you or Dask generates the dictionary):

A graph dictionary consists of:

1) keys: the identifier for a data entity of either type `str` or a tuple
    `(str, int)`.
2) values: a specification of what data entities this entity depends on and
    what to do with those entities (what operations to apply) to compute this
    particular data entity.

Dask data entity strings take the form of '<name_of_operation>-<unique_hash>'.

The key, as mentioned above, can either be a string or a tuple of a string and
and integer. The tuple comes into play, for example, when you are applying a map
operation to a function - multiple calls to your function have generated new
data entities, but the operations that have been applied to generate all them
were the same (though the input data for each call was different). Expounding
upon the map example, given a function `compute()` mapped onto a 3 item list of
data entities, dictionary entries will look like:

```
{
    ('compute-<hash>', 0): <the operation on the first list item>,
    ('compute-<hash>', 1): <the operation on the second list item>,
    ('compute-<hash>', 2): <the operation on the third list item>,
}
```

The hash for all 3 entries is the same. Read more:
http://docs.dask.org/en/latest/custom-collections.html#implementing-deterministic-hashing

You can see here that the integer portion of the (str, int) tuple corresponds to
a unique identifier for the n-th call of your operation.

In the case where your function is just called once, your dictionary entry for
this entity would simply be `'compute-<hash>': <operation>`, with no tuple.

The graph dictionary values - the dependencies of a data entity + the operations
to calculate it, are a bit more complicated. They can either consist of
(multiple levels of) Dask "tasks" or sometimes actual input data (itself, not a
string / `(str, int)` `tuple` identifier). An example of when actual data is
passed is when you create a Dask `Bag` from a sequence:

```
import dask.bag as db

filenames = ['raw_data/chelsea.jpg', 'raw_data/astronaut.jpg']

graph_dict = db.from_sequence(filenames).__dask_graph__()
```

`graph_dict` would look something like:

```
('from_sequence-638ecf64d497eff741f573e0d74b03a7', 0):

    list | ['raw_data/chelsea.jpg']

('from_sequence-638ecf64d497eff741f573e0d74b03a7', 1):

    list | ['raw_data/astronaut.jpg']
```

Note that Dask `Bag`s rely on the concept of partitions - the items in the bag
are divided up into partitions which are then individually sent off for
computation as the argument to your computation function for
`(number of partitions)` calls. If there were 500 filenames, for example, you
might end up with a bag with 100 partitions, each containing 5 of those
filenames. That is why you see that the values in `graph_dict` are lists even
though each list has only one member. Don't get mislead into thinking that there
are two `from_sequence` keys in our dictionary because we had two filenames.
Dask just chose to divide our bag into two partitions, each generating its own
data entity after computation.

What does a dictionary value for an operation look like?

Graph dictionary values for operations are tuples of (potentially nested) Dask
"tasks". Tasks are simply a name for a function and its particular arguments it
will use to compute its data entity. A task takes the form of:

`tuple` `(<function>, arg1, arg2, ...)`

Each argument is NOT actually data itself, but a data entity key in your graph
dictionary.

More concretely, for a computation that uses the following function:

```
def compute(filename):
    with open(filename, 'rb') as fhdl:
        return fhdl.read()
```

a graph dictionary might look like:

```
{
    'file_entity-13deadbeef14123': 'the_jungle.txt',
    'compute-3152baadf00d1038': (compute, 'file_entity-13deadbeef14123'),
}
```

This structure gets a bit more fun when you're using Dask functions on top of
your own. For example, let's say we want to `compute()` on multiple files by
`map`ing our function:

```
import dask.bag as db

filenames = ['raw_data/chelsea.jpg', 'raw_data/astronaut.jpg']

graph_dict = db.from_sequence(filenames).map(compute).__dask_graph__()
```

The `graph_dict` tasks now need to be nested - the final graph dictionary in
this example as put together by Dask on your mapping looks like:

```
{
    ('from_sequence-74d30dd097bc65e3bbc55700e653b181', 0):
        ['raw_data/chelsea.jpg'],
    ('from_sequence-74d30dd097bc65e3bbc55700e653b181', 1):
        ['raw_data/astronaut.jpg'],
    ('map-compute-669e5b87af812913f566cd24aab9ff4d', 0):
        (
            <function reify at 0x7f6a6801f510>,
            (
                <function map_chunk at 0x7f6a6801f840>,
                <function compute at 0x7f6a5aa6e9d8>,
                [('from_sequence-74d30dd097bc65e3bbc55700e653b181', 0)],
                None,
                {}
            ),
        ),
    ('map-compute-669e5b87af812913f566cd24aab9ff4d', 1):
        (
            <function reify at 0x7f6a6801f510>,
                (
                    <function map_chunk at 0x7f6a6801f840>,
                    <function compute at 0x7f6a5aa6e9d8>,
                    [('from_sequence-74d30dd097bc65e3bbc55700e653b181', 1)],
                    None,
                    {},
                ),
        ),
}
```

Pretty-printed, our structure looks like:

```
('from_sequence-638ecf64d497eff741f573e0d74b03a7', 0):

    list | ['raw_data/chelsea.jpg']

('from_sequence-638ecf64d497eff741f573e0d74b03a7', 1):

    list | ['raw_data/astronaut.jpg']

('map-compute-52043258a138bd4a450cfb184074f358', 0):

  Dask task:

      0 | function 	|   <function reify at 0x7fe5fdc518c8>
      1 | Dask task:

          0 | function 	|   <function map_chunk at 0x7fe5fdc51bf8>
          1 | function 	|   <function compute at 0xdeadbeef>
          2 | list 	|   [('from_sequence-638ecf64d497eff741f573e0d74b03a7', 0)]
          3 | NoneType 	|   None
          4 | dict 	|   {}

('map-compute-52043258a138bd4a450cfb184074f358', 1):

  Dask task:

      0 | function 	|   <function reify at 0x7fe5fdc518c8>
      1 | Dask task:

          0 | function 	|   <function map_chunk at 0x7fe5fdc51bf8>
          1 | Stage 	|   <function compute at 0xdeadbeef>
          2 | list 	|   [('from_sequence-638ecf64d497eff741f573e0d74b03a7', 1)]
          3 | NoneType 	|   None
          4 | dict 	|   {}
```

Breaking the `compute` operation down, we have:

* A Dask task consisting of the Dask `reify` function, which is itself passed a
    task.
* The passed task consisting of the Dask `map_chunk` function. This function
    is passed 4 arguments: our `compute` function, a partition of our bag, and
    (empty) additional `map_chunk` arguments. The empty `dict`, in this case,
    would have been any keyword arguments that we wanted to pass to `compute`
    through calling `.map(compute, kwarg_1='blah', kwarg_2='blarg')`.

In this interesting case, our `compute` call is not actually directly a "task"
in this graph. The `map_chunk` task will take each filename element of the
list in its bag partition argument and apply it to our `compute`.

Notice that there are two entries in our graph dictionary for this `compute`
operation - one for each partition of our bag. Therefore, the structure is
essentially: for each partition, `reify` the results of `map`ing each filename
element in it onto the `compute` function. Because this is the same task applied
multiple times to different data entities (partitions), each data entity key
coming from this task takes the form of a tuple with a unique integer identifier
for each partition.
